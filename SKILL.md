---
name: alipay-payment-integration
description: 给任意项目接入支付宝网页支付（电脑/手机网站支付），含下单、异步回调验签、幂等发货、原路退款与验收。用户说「接入支付宝」「加个支付」「收款」「支付回调」「退款」「支付宝签名/验签报错」时使用。内容来自一个真实上线并跑通真钱（付款+退款）的项目，沉淀了 13 条实测踩坑。
---

# 支付宝网页支付接入（可直接照抄）

> 本 skill 来自一个**真实上线并跑通真钱**的项目（下单 → 收银台 → 异步通知 → 验签 → 发货 → 原路退款，
> 9.9 元真实支付 + 真实退款验证过）。下面每条「坑」都是实际踩出来的，不是推演。

## 0. 先问清楚这四件事（问不出来别动手）

1. **主体资质**：企业或个体工商户营业执照（个人账号可签但额度受限；无执照只能线下）
2. **网站已 ICP 备案**，且**备案主体与签约主体一致**（不一致需授权函）
3. **每个站一个「网页应用」** —— 同一支付宝账号可以建多个应用，但**一个应用对应一个网站地址**；
   多站复用同一 APPID 容易被审核驳回。**每个应用一套密钥**，不要混用。
4. 商品/服务描述与「经营类目」一致（否则审核驳回）

## 1. 配置项（少一个都跑不通）

```bash
ALIPAY_APP_ID=            # 开放平台应用详情页复制，16 位数字（读截图必错，见坑 1）
ALIPAY_PRIVATE_KEY=       # 应用私钥（PKCS#8 或 PKCS#1，裸 base64 或带 \n 都行）
ALIPAY_PUBLIC_KEY=        # ★ 支付宝公钥（不是应用公钥！）
ALIPAY_GATEWAY=https://openapi.alipay.com/gateway.do   # 沙箱时换成沙箱网关
ALIPAY_NOTIFY_URL=        # 可选，默认 ${对外域名}/api/payment/alipay/notify
ALIPAY_RETURN_URL=        # 可选，默认 ${对外域名}/account
```

**去哪拿**：
- APPID / 应用私钥 / 应用公钥：开放平台 → 控制台 → 应用 → **开发设置 → 接口加签方式（密钥）**
- **支付宝公钥**：**先上传应用公钥并保存**，弹窗下方才会出现「支付宝公钥」（也可「下载支付宝公钥」）
- 收款账号 PID（可选，配了就一起校验）：商家平台 → 账号信息

## 2. ★★★ 签名规则：请求与通知**不一样**，搞反必挂

| 场景 | 待签串包含 `sign_type` 吗 |
|---|---|
| **请求签名**（下单/查询/退款）| **包含** —— 除 `sign` 外**所有**参数都参与 |
| **异步通知 / 同步跳转验签** | **不包含** —— 除 `sign` **和** `sign_type` 外参与 |

```ts
// Node / TypeScript
function signContent(params: Record<string,string>, opts: { excludeSignType?: boolean } = {}) {
  const ex = opts.excludeSignType ?? false;
  return Object.keys(params)
    .filter(k => k !== "sign" && (!ex || k !== "sign_type") && params[k] !== undefined && params[k] !== "")
    .sort()                       // ASCII 升序
    .map(k => `${k}=${params[k]}`) // ★ 用**原始值**，不要 URL 编码后再签
    .join("&");
}
// 请求：signContent(params)                        ← 含 sign_type
// 验通知：signContent(params,{excludeSignType:true}) ← 不含
```

**坑 2（最费时间的一个）**：把 `sign_type` 也排除后再签请求，网关**只回一句 `invalid-signature`**，
文案还是「建议检查签名私钥与应用公钥是否匹配」—— 会把人带去查密钥（我因此白折腾一轮，还让用户改了公钥）。
**决定性诊断法**：把错误页里「网关生成的验签字符串为：…」抓下来，反复解 `&amp;` 多层转义后
与本地签名串**逐字符比对**，差异点直接指出哪一段不一致。

**坑 3**：`invalid-app-id` 与 `invalid-signature` 是两件事 —— 前者是 APPID 错（常见于**读截图少一两位**），
后者才轮到密钥/签名。**APPID 一律复制粘贴，不要手打或识图。**

**坑 4**：应用**未上线**时网关回 `not-online-app`（说明签名已通过）→ 去开放平台「上线/提交审核」；
产品**未签约**时回「无权限使用该接口」→ 去 b.alipay.com 产品中心签约。两件事都要做，缺一不可。

**坑 5**：`ALIPAY_PUBLIC_KEY` 填成**应用公钥**是最常见的配置错。两者都是 `MIIBIjANBgkq...` 开头。
自检：用私钥推导出应用公钥，若与配置的「支付宝公钥」**相同**，就是填错了。

## 3. 下单

```ts
// 电脑网站支付（支付宝收银台，扫码/登录支付）
method: "alipay.trade.page.pay",  biz.product_code: "FAST_INSTANT_TRADE_PAY"
// 手机网站支付（手机浏览器，直接唤起支付宝 App）
method: "alipay.trade.wap.pay",   biz.product_code: "QUICK_WAP_WAY"
// 金额：★ 单位是「元」，字符串，两位小数（"9.90"）—— 与微信的「分」相反
biz_content = JSON.stringify({ out_trade_no, total_amount: "9.90", subject, product_code })
```

拼成 `https://openapi.alipay.com/gateway.do?<params>&sign=<urlencode(sign)>` 后**前端跳转**即可。

**坑 6（只有真下单才会暴露）**：`return_url` **必须用对外域名**。
在 nginx/反代后面用 `new URL(req.url).origin` 会拿到 `http://0.0.0.0:3000` ——
用户付完款跳到内网地址白屏，且看起来像"支付把账号弄丢了"。用 `BETTER_AUTH_URL` / `SITE_URL` 这类
显式配置的对外地址。

## 4. 异步回调（核心，安全边界都在这）

```ts
POST /api/payment/alipay/notify
const raw = await req.text();                       // form-urlencoded
const params = Object.fromEntries(new URLSearchParams(raw));

// ① 验签（不通过必须硬失败，不能"验不过就当没验过"）—— 排除 sign_type
if (!verifyRSA2(signContent(params, { excludeSignType: true }), params.sign, ALIPAY_PUBLIC_KEY)) {
  throw new Error("验签失败");
}
// ② app_id 必须等于自己的（防别人的应用把回调打到你这）
if (params.app_id !== ALIPAY_APP_ID) throw new Error("app_id 不匹配");
// ③ 金额必须与订单一致（★ 元 → 分，避免浮点：Math.round(parseFloat(x) * 100)）
if (yuanToCents(params.total_amount) !== order.amountCents) throw new Error("金额不一致");
// ④ trade_status 只认 TRADE_SUCCESS / TRADE_FINISHED
// ⑤ 幂等：条件 UPDATE 认领，**不要"先读状态再无条件 UPDATE"**
const claimed = await db.update(order)
  .set({ status: "paid", tradeNo: params.trade_no, paidAt: new Date() })
  .where(and(eq(order.outTradeNo, params.out_trade_no), inArray(order.status, ["pending","closed"])))
  .returning();
if (claimed.length === 0) return new Response("success");   // 已被别的回调处理过
await fulfill(claimed[0]);                                   // 发货（发次数/开月卡）
return new Response("success");  // ★ 必须纯文本 success，否则支付宝会一直重试
```

**坑 7**：幂等第二层必须是**条件 UPDATE 认领**。写成"查状态 → 判断 → UPDATE"在并发下会重复发货
（实测同一单被发了 5 次）。条件里要用**积极白名单**（`pending`/`closed`），
**绝不能让 `refunded` 被改回 `paid`**。
**坑 8**：`closed`（超时关闭）之后用户仍可能付款成功（支付宝允许补付），所以 `closed` 也要认领。
**坑 9**：回调报文可能重复、乱序、延迟几分钟到几小时 —— 所有逻辑都要能重放。

## 5. 原路退款

```ts
method: "alipay.trade.refund"
biz_content = { out_trade_no, refund_amount: "9.90", out_request_no: "RF-<订单号>" }
```

**坑 10（顺序）**：**先调渠道退款，成功了再写库标记 `refunded`**。
反过来先改状态，一旦渠道失败这笔单就再也退不了（`refunded` 是终态）。
**坑 11**：退款单号 `out_request_no` 要**确定性推导**（如 `RF-<out_trade_no>`），
这样重试不会退成两笔；随机单号 = 可能退两次。
**坑 12**：同步返回里 `fund_change === "Y"` 才代表钱真的动了；`refund_fee` 可能是"本次退款金额"而不是累计。
**坑 13**：回收权益要 `GREATEST(0, balance - n)` 夹住 —— 用户可能已把次数花掉，
扣成负数等于"欠费"，比"扣光"更糟。

## 6. 生产守卫

- **mock/沙箱渠道在生产必须硬失败**（`NODE_ENV === "production"` 时直接抛错，而不是降级）
- 未配置的渠道**不出现**在支付方式列表里，**不要**给出一个点了会报错的按钮
- 价格与商品**只从服务端取**（DB / 常量表），前端只传 `productCode`；客户端传来的金额一律不信
- 回调地址用 HTTPS 且与站点同域

## 7. 验收清单（缺一条都不算"接好了"）

```
[ ] 下单接口返回的 payUrl 能打开真收银台，且金额/商品名正确
[ ] payUrl 里 notify_url、return_url 都是对外域名（不是 0.0.0.0 / localhost）
[ ] 真付一笔小额 → 回调日志出现「验签通过 + 已发货」
[ ] 同一单回调重放多次 → 只发一次货（查流水条数，不是查余额）
[ ] 付完刷新页面 → 权益真的到账（不是只有日志说成功）
[ ] 原路退款 → fund_change=Y，状态 refunded，权益按要求回收且不为负
[ ] 用错误签名打回调 → 服务端必须拒绝（别只测"正确能过"）
[ ] 金额篡改（回调里把 9.90 改成 0.01）→ 必须拒绝
```

## 8. Python 版本要点（非 Node 项目）

```python
# 签名：RSA-SHA256 + PKCS#1 v1.5
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding
sign = base64.b64encode(key.sign(content.encode("utf-8"), padding.PKCS1v15(), hashes.SHA256())).decode()
# 验签（通知）：public_key.verify(sig, content.encode(), PKCS1v15(), SHA256())
# content 构造规则与上面完全一致：排序 / 去空 / 原始值 / 请求含 sign_type、通知不含
```

## 9. 复用时的三条纪律

```
① 同一个支付宝账号可以建多个「网页应用」，但**一个应用对应一个网站地址**
   → 新项目要**新建应用并单独签约**，不要复用别人的 APPID（审核会看网站地址与经营内容）
② 每个应用**一套密钥**（应用私钥 / 支付宝公钥各自独立），不要混用
③ 密钥只写进该项目的 .env（600 权限），不进仓库；本文件里也不放任何真实密钥
```

> 本机主人的账号与已有应用信息记在**同目录的 `ACCOUNT.local.md`**（刻意不发布、不入库）。

