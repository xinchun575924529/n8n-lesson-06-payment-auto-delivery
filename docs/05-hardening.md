# 05 · 生产加固清单

修复版解决了「能跑对」，上线前还要解决「跑得稳、不被嫖、出事有感知」。

## 1. 支付验签（必修，否则等于敞开金库）

- **Stripe**：用 `endpointSecret` 校验 `Stripe-Signature` 头（`t=时间戳,v1=HMAC-SHA256`），Code 节点 15 行可写完；时间戳偏差 >5 分钟直接拒
- **支付宝当面付**：notify_url 回调用**支付宝平台公钥** RSA2 验签，且验 `out_trade_no` 与本地订单一致、`trade_status=TRADE_SUCCESS`（docs/06）
- 通用原则：**webhook 入口白名单事件类型**，非预期 type 直接 200 吞掉不处理（防探测）

## 2. 交付匹配从「phone 单字段」升级为「多字段 + 显式兜底」

- 付款链接携带 `client_reference_id`（渠道 + chat_id/phone 哈希），支付成功事件用它精确匹配
- 匹配不上**必须**走 Fallback 人工通知，并把原始事件落盘留证——静默 = 客诉 + 退款纠纷
- 台账加唯一约束（phone+channel），重复咨询只更新不追加（修复版已实现）

## 3. 断链感知：所有查询节点开 alwaysOutputData + 失败告警

- 任何「查表后接动作」的节点，0 item 都要有显式分支（告警/兜底），不允许链路无声终止
- 加一条兜底 Error Workflow：任何执行失败 → 飞书/邮件通知运维

## 4. 幂等与防重

- 支付平台会重发 webhook（未秒回 2xx 时），用 `event_id` / `out_trade_no` 去重，已交付的直接 200 返回
- 发货动作本身幂等：重发同一链接无害，但「授权码发放」类要先查是否已发过

## 5. 一个 path 一个职责

- webhook path 不复用、不靠 method 区分业务（坑#6）；命名带业务前缀，Meta/Stripe/支付宝后台回调地址各配各的

## 6. 密钥管理

- 应用私钥（Stripe secret / 支付宝 RSA2 私钥）只存 n8n 凭证或环境变量，**绝不写进工作流 JSON、绝不进 Git**
- 支付宝应用公钥配到开放平台后要回拿「支付宝公钥」用于验签，两把钥匙别搞混（应用公钥=你给它看门牌，支付宝公钥=你验它的信）

## 7. 状态机完整性

- 台账 status 流转：`new → sent_link → paid → delivered`；催单只捞 `sent_link`，交付后推进到 `delivered`，防止重复发货
- 每日对账：支付平台账单 vs 台账 delivered 记录，差异入告警