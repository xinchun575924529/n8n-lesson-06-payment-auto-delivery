# 02 · 坑集（6 个实锤坑，全部经断言/双跑对照复现）

## 坑#1：Check Payment Validity 连线接反——交付逻辑整体反转【最重磅】

**现象**：Check Payment Validity 节点，`skip=true`（无效付款/无手机号）的**真分支**被连到了「Retrieve Client From Sheets（正常交付链）」，假分支才接「Send Fallback Notification」。按模板字面接线：**无效付款会走进交付链、有效付款被当成无效发人工通知**——整个交付逻辑反转。

**实锤**（B 关忠实版双跑）：投喂一笔完全有效的付款（手机号齐全、金额正常），outbox 里出现的不是发货消息，而是 `fallback` 人工通知；客户付了钱，永远收不到货。

**修复**（fixed 版已做）：把两个输出分支的连线对调，`skip=false`（有效）走交付链、`skip=true`（无效）走 Fallback。修复版断言 S4：有效付款正确查表→识别 whatsapp 渠道→发货。

## 坑#2：Retrieve Client From Sheets 无 alwaysOutputData——查无买家静默断链

**现象**：查无此买家时 Google Sheets 节点输出 **0 个 item**，下游「Determine Communication Channel」等所有节点**全部不执行**，买家永远收不到货且**无任何告警**。对照：模板里 Retrieve Unpaid Clients 恰好开了 `alwaysOutputData`——作者知道这个特性，却在最关键的交付查询上漏开。

**实锤**（A 关单元 + B 关忠实版）：无 phone 的付款事件进入查表后，outbox 零新增、台账零变动、执行状态照样 success——钱收了，发货链无声死亡。

**修复**（fixed 版已做）：查表节点开 `alwaysOutputData`（替身 Code 版返回带空标记的 item），查无买家 → 显式走 Fallback 通知人工，绝不静默。

## 坑#3：lookupValue 多余 `==` 前缀——筛选失配

**现象**：Retrieve Client From Sheets 的 lookupValue 写成 `=={{ $json.phone }}`，带着多余的 `==` 前缀。Sheets 筛选按字面匹配，手机号对不上 `==8613...`，查表永远 0 命中——叠加坑#2，所有付款全部静默丢单。

**实锤**（B 关忠实版断言 F4）：投喂有效付款，outbox 里没有任何含该手机号的发货记录；修复版去掉前缀后同一投喂立即命中发货。

**修复**（fixed 版已做）：lookupValue 改为 `{{ $json.phone }}`。

## 坑#4：Stripe webhook 无验签——任何人可白嫖发货

**现象**：模板对 `stripe-payment` webhook 收到的事件**零校验**——没有验证 `Stripe-Signature` 头。任何人 POST 一个伪造的 `checkout.session.completed` payload（填上目标手机号）即可白嫖发货。

**实锤**：B 关所有测试 payload 都是裸 POST，无签名也全部被接受执行——这本身就是漏洞复现。

**修复**（fixed 版 + docs/05）：生产必须验签。用 Stripe 官方库的 `constructEvent(payload, sig, endpointSecret)` 或在 Code 节点 HMAC-SHA256 校验 `t,v1` 签名对；替身版在 Extract 节点加了事件类型白名单（非 `checkout.session.completed` 只回 skip）。支付宝方案同理：notify_url 回调必须 RSA2 验签（docs/06）。

## 坑#5：phone 单字段脆弱匹配——TG 客户永远查不到

**现象**：交付匹配只靠 `phone` 一个字段。而 Telegram 客户登记时根本没有 phone（TG 消息里只有 chat_id），除非 Stripe checkout 页**强制填手机号**且客户填的与咨询时留的**完全一致**，否则永远查不到买家 → 叠加坑#2 直接静默失败。

**修复**（fixed 版 + docs/05）：付款链接里带上渠道与 chat_id 参数（如 `?client_ref=tg_88001`），付款事件用 `client_reference_id` 匹配而非裸手机号；至少做到「手机号 + 渠道」双字段匹配，匹配不上走 Fallback。

## 坑#6：催单 webhook 与 WA intake 共用 path——极易配混

**现象**：催单/验证回执（GET，应答 `hub.challenge`）与 WhatsApp intake（POST 事件）共用 path=`whatsapp-verify`，只靠 HTTP method 区分。配置 Meta 后台回调、排查问题时极易混淆，改错一边另一边就 404。

**修复**（fixed 版已做）：拆成独立 path（`x-18042-reminder` 专管验证/催单），并在课程里强调「一个 path 一个职责」原则。