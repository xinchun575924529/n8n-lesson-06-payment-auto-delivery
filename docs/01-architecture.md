# 01 · 架构拆解

## 节点清单（9 功能节点 + 11 便签）

| # | 节点 | 类型 | 作用 |
|---|------|------|------|
| 1 | When WhatsApp Message Received | webhook | 收 WhatsApp 云 API 事件（path=`whatsapp-verify`, POST） |
| 2 | When Telegram Message Received | webhook | 收 Telegram webhook（path=`telegram-incoming`） |
| 3 | When Stripe Payment Posted | webhook | 收 Stripe event（path=`stripe-payment`） |
| 4 | When Payment Reminder Triggered | webhook(v2.1) | 催单触发 / Meta 验证回执（应答 hub.challenge，GET） |
| 5 | Extract WhatsApp Message / Extract Telegram Message / Extract Stripe Payment Details | code ×3 | 解析三路 payload：WA→phone+message；TG→chat_id+first_name；Stripe→phone/amount/币种 |
| 6 | Check … Validity ×3 / Determine Communication Channel / Check Payment Status Pending | if ×5 | 三路有效性校验、渠道识别、催单状态分支 |
| 7 | Add WA/TG Client to Sheets、Retrieve Client From Sheets、Retrieve Unpaid Clients | googleSheets ×4 | 客户台账读写（替身版=本地 JSON） |
| 8 | Send WA Payment Link、Deliver PDF via WhatsApp、Send Fallback Notification | httpRequest ×3 | 走 Facebook Graph API v19 发 WhatsApp 文本 |
| 9 | Send Telegram Payment Link、Deliver PDF via Telegram | telegram ×2 | 发 TG 付款链接 / PDF 链接 |
| 10 | Acknowledge ×4 | respondToWebhook ×4 | 回 OK / 回显 hub.challenge |

## 四条链路

```text
链路1 WA 咨询：x-wa-in → Extract WA → Check Validity ─有效→ Add WA Client
              → Send WA Payment Link → Acknowledge
链路2 TG 咨询：x-tg-in → Extract TG → Check Validity ─有效→ Add TG Client
              → Send TG Payment Link → Acknowledge
链路3 付款交付：x-stripe-in → Extract Stripe → Check Payment Validity ─有效→
              Retrieve Client From Sheets → Determine Channel ─┬─WA→ Deliver via WA
                                                              └─TG→ Deliver via TG
              Check Payment Validity ─无效→ Send Fallback Notification
链路4 催单：x-reminder → Retrieve Unpaid Clients → Check Status Pending
              ─sent_link→ Build Reminder → Post WA Reminder
```

## 数据流关键设计

- **phone 是跨链路的唯一 join key**：咨询链登记 phone，付款链用 Stripe 回传的手机号反查。这意味着「checkout 页必须强制填手机号且客户要填对」——脆弱点，见坑#5
- **status 状态机**：台账行 `status` 从空 → `sent_link`（已发链接）→ 付款后 delivery。催单链只捞 `sent_link` 状态的行
- **channel 字段决定交付渠道**：`whatsapp` / `telegram`，Determine Communication Channel 按它分流
- **webhook 应答一律 respondToWebhook**：Meta/Stripe 都要求快速 2xx，否则重试风暴

## 版本混用提醒

模板 googleSheets 节点 typeVersion 4 与 4.7 混用、webhook 1 与 2.1 混用，导入较新 n8n（2.x）可能有兼容性警告，属正常现象，修复版已统一为标准节点。