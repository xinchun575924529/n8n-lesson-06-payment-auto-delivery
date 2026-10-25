# 00 · 总览

## 业务场景

你在卖数字产品（PDF 教程、素材包、软件授权码……），客户从 WhatsApp 或 Telegram 找上门咨询。模板把「咨询 → 收款 → 发货」整条链路自动化：

1. 客户发消息咨询 → webhook 收事件 → Code 节点解析出手机号/chat_id
2. 客户登记进 Google Sheets 台账 → 通过原渠道回发 Stripe 付款链接
3. 客户付款 → Stripe `checkout.session.completed` webhook 触发
4. 按付款时留的手机号查台账，识别这个客户是从 WhatsApp 还是 Telegram 来的
5. **通过原渠道**把数字产品下载链接发过去——WA 来的回 WA，TG 来的回 TG
6. 支线一：定时查「已发链接但还没付款」的客户 → 发催单提醒
7. 支线二：付款事件缺手机号（无法匹配客户）→ 发人工处理通知（Fallback）

## 为什么值得学

- 这是「**收款后自动发货**」这一跨境/数字产品售卖最经典场景的完整骨架：IM 双渠道 intake + 支付 webhook + 台账匹配 + 原路交付，换个皮就是课程售卖、素材站、授权码发放
- 模板拿到 n8n 官方库上架、看起来能跑，但**实锤 6 个坑**，其中「连线接反」和「静默断链」任何一个都能让你「收了款不发货还毫无察觉」——是绝佳的生产级排错教材
- 凭证全可替身：WhatsApp 企业 API / Google Sheets / Stripe 三样国内都难落地，本课给出全套免费替身 + 支付宝当面付本地方案

## 关键概念词汇

| 词 | 含义 |
|---|---|
| WhatsApp Cloud API | Meta 官方的 WhatsApp 企业消息接口（需企业认证，国内基本不可用；本课 mock） |
| checkout.session.completed | Stripe 结账完成事件，webhook 推送给商户 |
| 当面付 | 支付宝线下/扫码收款产品，个体工商户可签约，支持异步回调 notify_url |
| 台账 | 本课对客户登记表的称呼：模板用 Google Sheets，替身版用本地 JSON 文件 |
| Fallback | 兜底分支：付款无法匹配客户时通知人工处理，而不是静默吞掉 |
| RSA2 加签 | 支付宝开放平台的接口签名机制（SHA256withRSA），回调验签防伪造 |