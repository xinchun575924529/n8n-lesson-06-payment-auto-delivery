# 动手练习（含答案）

## 练习 1：找出「收了款不发货」的真凶

学员反馈：Stripe 后台能看到付款成功，客户（WhatsApp 渠道，付款时填了手机号）却没收到 PDF 链接，n8n 执行记录显示「success」。按坑集列出排查顺序，并说明每步怎么验证。

<details><summary>答案</summary>

1. **先查坑#3（lookupValue `==` 前缀）**：看 Retrieve Client From Sheets 的 lookupValue 是否带 `==`；验证=看该节点输出 item 数，0 = 失配
2. **再查坑#2（无 alwaysOutputData）**：查表 0 命中时下游是否全部没执行；验证=看执行详情里 Determine Communication Channel 有没有运行记录，没有=静默断链实锤
3. **再查坑#1（连线接反）**：看 Check Payment Validity 的两个输出分别接到哪；验证=看这笔执行走了 Fallback 还是交付链——执行 success 但走 Fallback 就是接反
4. **最后查坑#5（phone 匹配）**：Stripe 回传的 phone 与台账登记的是否逐字符一致（国家码、空格、+号）；验证=对比两边原始字符串

注意「执行 success ≠ 发了货」——这正是静默类坑最危险的地方。

</details>

## 练习 2：给修复版加「发货后状态推进」

修复版台账 status 目前只到 `sent_link`，付款发货后没有推进。请设计改动：让 S4 场景交付成功后该客户 status 变为 `delivered`，且重复收到同一付款事件时不重复发货（幂等）。

<details><summary>答案</summary>

1. 在 Retrieve Client From Sheets 命中后、Deliver 节点前，加一个 Code 节点：把台账中该客户行 `status` 改为 `delivered`、记录 `paid_at` 与该支付事件 ID（如 Stripe `event.id` / 支付宝 `out_trade_no`）
2. 幂等：Extract Stripe Payment Details 后先查「该 event_id / out_trade_no 是否已处理过」（台账加 `processed_events` 数组或单独文件），已处理 → 直接 respondToWebhook 回 200 结束
3. 顺序必须「先标记已处理再发货」或同事务，防止发货成功但标记失败导致重发
4. 验证：驱动脚本里对同一 STRIPE_OK payload 连发两次，outbox 中 deliver 只应出现 1 次

</details>

## 练习 3：把 Stripe 链替换成支付宝当面付

按 docs/06，写出替换 Stripe webhook 链需要改动的节点清单与每个节点的核心改动点（不需要写完整代码）。

<details><summary>答案</summary>

1. **付款链接节点**（Send WA/TG Payment Link）：从「发 Stripe checkout URL」改为「调 alipay.trade.precreate 预下单 → 发 qr_code 扫码链接」；`out_trade_no` 编码客户标识（渠道+chat_id+日期序号）
2. **When Stripe Payment Posted webhook**：path 改为支付宝 notify_url 专用路径；注意支付宝回调是 **form-urlencoded POST**，解析方式与 JSON 不同
3. **Extract Stripe Payment Details → Extract Alipay Notify**：验签（支付宝公钥 RSA2）→ 验 out_trade_no 在台账存在、金额一致、trade_status=TRADE_SUCCESS → 从 out_trade_no 解码客户标识
4. **Check Payment Validity**：判定条件改为「验签通过 && 金额一致 && 订单存在」
5. **Retrieve Client From Sheets**：lookup 键从 phone 改为 out_trade_no 解码出的客户标识（修掉坑#5）
6. **Acknowledge**：支付宝要求回纯文本 `success`，不是 JSON
7. 台账 status 机与幂等处理同练习 2

</details>