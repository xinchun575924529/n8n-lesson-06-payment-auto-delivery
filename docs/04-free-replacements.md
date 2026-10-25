# 04 · 免费替身方案（0 元跑通全课）

| 模板需要 | 缺口评估 | 替身方案 |
|---|---|---|
| WhatsApp Cloud API（Meta 企业号 phone_number_id + access token） | ❌ 需 Meta 开发者 + 企业认证，国内基本不可用 | **mock**：Code 节点把「应发送的消息」追加写入本地 `outbox.jsonl`，教学完全等价；生产可换企业微信/飞书机器人 |
| Telegram bot token | ✅ 免费，BotFather 一分钟立得 | 真实可用，不需替身；课程里 TG 链可全程真跑 |
| Google Sheets OAuth2 | ⚠️ 需 Google 账号 + API 授权，繁琐 | **本地 JSON 台账** `clients.json`：Code 节点读改写，与 Sheets 语义一一对应（append / lookup / filter） |
| Stripe webhook + 付款链接 | ⚠️ 测试模式可用，但国内收款受限 + webhook 需公网回调 | **mock payload 直打本地 webhook**；真实收款换**支付宝当面付**（docs/06） |
| PDF 下载链接 | — | 任意占位 URL / 对象存储直链 |

## 替身实现要点

- **本地 JSON 台账**：修复版用 Code 节点实现三个操作——`add`（查重后 append）、`retrieve`（按 phone 精确匹配，无命中时返回带空标记 item 而非 0 item，修坑#2）、`retrieve_unpaid`（filter status=='sent_link'）。文件读写用 Node `fs`，路径用环境变量/参数化，避免硬编码
- **outbox mock 发送**：每条「发送动作」落一行 JSONL：`{kind: link_wa|link_tg|deliver_wa|deliver_tg|reminder_wa|fallback, data: {...}, ts}`。断言直接读 outbox，比肉眼盯 IM 更可靠，这也是本课驱动脚本的断言基础
- **mock Stripe payload**：直接按 `checkout.session.completed` 的 JSON 结构 POST 到本地 webhook（课程驱动脚本内置 3 种：正常/无 phone/未知号码），全程离线
- **Telegram 真跑（可选加分项）**：有 TG bot token 的学员，把 mock 发送节点换回 telegram 节点、payload 换成真 chat_id，即可体验真实交付——这是本课唯一可 0 元真跑的 IM 渠道

## 为什么这样设计

教学目标是「骨架 + 坑 + 修复」，不是「教你申请 Meta 企业认证」。把所有要钱要资质的依赖全部参数化替身后，学员 10 分钟就能在本机把四条链路全部点亮，把精力集中在真正值钱的部分：**理解模板为什么会静默丢单、如何系统性地防**。