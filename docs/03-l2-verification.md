# 03 · L2 验证报告（全部实跑，可复现）

## A 关 · 单元级（2026-09-16，27/27 全过）

工作流 `L2A-UnitTest-18042-SalesPDFBot`，忠实移植 3 个 Extract Code 节点 + 5 个 IF 分支逻辑，CLI n8n 2.33.3 execute：

| 单元 | 断言数 | 要点 |
|---|---|---|
| Extract WhatsApp Message | 4 | phone trim / message lower·trim / 状态回执无 messages 不炸 / 畸形 body 不炸 |
| Extract Telegram Message | 4 | chat_id 数字→字符串、first_name 缺省 'friend'、无 text 不炸、畸形 body 不炸 |
| Extract Stripe Payment Details | 6 | 正常成交解析、amount/100、币种大写、非 completed 事件只回 skip、无 phone/无 data 兜底 |
| IF 分支逻辑 ×4 | 10 | WA/TG 有效性（isNotEmpty）、渠道识别、Payment Validity 路由如实复刻、Pending 双路 |
| Set Build Payment Reminder | 3 | 字段映射、ISO timestamp、不带 first_name |

**合计 27/27 全过**，并在此关实锤坑#1（连线接反，IF7/IF8 如实复刻反日语义）与坑#2（0 item 断链）。

## B 关 · mock 全链双跑（2026-10-25 重跑，15/15 全过）

同一驱动脚本对**忠实版（含 bug）**与**修复版**各跑一遍，外部依赖全替身（本地 JSON 台账 + outbox 文件 mock 发送）：

### 忠实版——如实复现模板 bug（教学对照组）

| 断言 | 结果 |
|---|---|
| F1 [坑#3 复现] WA 新客静默丢单：无入库无链接 | ✅ |
| F2 WA 重复咨询也无催单（链条已死） | ✅ |
| F3 TG 新客正常入库 + 收到付款链接（TG 链设计可用） | ✅ |
| F4 [坑#2/3 实锤] 有效付款被「==」前缀 + 断链静默丢单 | ✅ |
| F5 [坑#1 实锤] 有效付款无人收货，进了 Fallback（IF 语义实测反转） | ✅ |
| F6 Reminder 验证回显 hub.challenge 可用 | ✅ |

忠实版实跑落证：台账仅 1 行（TG 客户），outbox 仅 `[link_tg, fallback]`——**WA 咨询与有效付款全部无声消失**。

### 修复版——7 场景端到端

| 断言 | 结果 |
|---|---|
| S1 WA 新客入库 1 次（status=sent_link）+ 付款链接仅发 1 次 | ✅ |
| S2 二次咨询触发催单（不重复发链接、不重复入库） | ✅ |
| S3 TG 新客入库 + 付款链接 | ✅ |
| S4 [修复验证] Alice 付款成功：查表命中 whatsapp 渠道 → 发货到手机 | ✅ |
| S5 无 phone 付款 → Fallback 人工通知（skip=true 走正确分支） | ✅ |
| S6 未知号码不发货（已知边界，查无此人显式标注） | ✅ |
| E1 无效消息不入库 | ✅ |
| S7 修复版验证回显 OK | ✅ |

修复版实跑落证：台账 2 行，outbox `[link_wa, reminder_wa, link_tg, deliver_wa, fallback]`——每条链路都有明确归宿。

**重跑方式**：启动本地 n8n（127.0.0.1:5678）导入 `workflow-fixed.json` 并激活，运行课程附带的驱动脚本（L2 阶段产出 `test-l2b2-18042.py`）即可复现以上全部断言。