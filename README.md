# 支付宝网页支付接入 Skill（AI Agent 用）

一个给 AI 编码助手（WorkBuddy / Claude Code / Cursor / Codex 等）用的 **Skill**，
让 agent 给任意项目接入支付宝网页支付时不必重新踩坑。

内容来自一个**真实上线并跑通真钱**的项目：真实支付 9.9 元 → 异步回调 → 验签 → 发货 →
原路退款，整条链路全部实测过，下面 13 条"坑"都是实际踩出来的，不是推演。

## 这个 Skill 解决什么

接入支付宝最容易挂的不是"调不通接口"，而是这四类：

1. **签名规则记反**：请求签名要**包含** `sign_type`，异步通知验签要**排除**它。
   搞反后网关只回一句 `invalid-signature`，文案还写着"建议检查私钥与应用公钥是否匹配"
   —— 会把人带去查密钥，白折腾几轮。
2. **幂等没做对**：回调会重复、乱序、延迟。写成"先读状态再无条件 UPDATE"在并发下**重复发货**。
   正确做法是条件 UPDATE 认领（且 `closed` 也要认领，用户能补付）。
3. **退款顺序反了**：先改状态再调渠道，一旦渠道失败这笔单就再也退不了（`refunded` 是终态）。
4. **只有正向用例**：不测"错误签名必须被拒""金额篡改必须被拒"，等于没测安全边界。

## 怎么装

**WorkBuddy / 通用**：把 `SKILL.md` 放进你的 skills 目录

```bash
mkdir -p ~/.workbuddy/skills/alipay-payment-integration
curl -o ~/.workbuddy/skills/alipay-payment-integration/SKILL.md \
  https://raw.githubusercontent.com/abcha3/alipay-payment-skill/main/SKILL.md
```

**Claude Code**：

```bash
mkdir -p .claude/skills/alipay-payment-integration
cp SKILL.md .claude/skills/alipay-payment-integration/
```

**Cursor / Codex / Windsurf**：把 `SKILL.md` 内容贴进对应的 rules 文件
（`.cursor/rules/`、`AGENTS.md`、`.windsurfrules`）。

装好后直接用自然语言说「给这个项目接入支付宝支付」/「加个支付回调」/「退款怎么接」，agent 会自动加载。

## 里面有什么

| 章节 | 内容 |
|---|---|
| 0. 前置四问 | 主体资质、ICP 备案主体一致、一应用一站点、经营类目匹配 |
| 1. 配置项 | 6 个环境变量 + 各自在后台哪个位置获取 |
| 2. **签名规则** | 请求 / 通知的对照表 + TS / Python 实现 + 正确诊断法 |
| 3. 下单 | page.pay / wap.pay 区别、金额单位（元/两位小数）、`return_url` 坑 |
| 4. 异步回调 | 验签硬失败、app_id 校验、金额校验、条件 UPDATE 幂等、返回纯文本 `success` |
| 5. 原路退款 | 先退钱再改状态、单号确定性推导、`fund_change`、权益回收 `GREATEST(0,…)` |
| 6. 生产守卫 | mock 生产硬失败、未配置渠道不出现在列表、价格只从服务端取 |
| 7. 验收清单 | 8 条，含三个反向用例 |
| 8. Python 要点 | 非 Node 项目的等价实现 |

## 适用边界

- 覆盖**电脑网站支付（page.pay）**与**手机网站支付（wap.pay）**，不含支付宝小程序、APP 支付、当面付
- 微信支付不在本仓库（签名体系完全不同，属于另一套）

## 许可

MIT，随便用、随便改、商用也行，不用署名。

> ⚠️ 本仓库**不含任何真实密钥或账号信息**。APPID / 私钥 / 支付宝公钥都要从你自己的
> 开放平台应用里取，写进你项目自己的 `.env`。
