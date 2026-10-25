# n8n 实战课 06：收款后自动发货机（Stripe→WhatsApp/Telegram 模板修复版 + 支付宝当面付本地方案）

> 官方模板：[Deliver digital products via WhatsApp and Telegram after Stripe payment](https://n8n.io/workflows/18042/)（模板 ID 18042，9 功能节点 + 11 张教学便签）

客户在 WhatsApp / Telegram 里咨询 → 工作流登记客户、回发付款链接 → 付款成功 webhook 触发 → 按手机号查客户台账、识别来源渠道 → 通过**原渠道**秒发数字产品下载链接。另带未付款催单链路和无效付款兜底。**理想很丰满，但模板实锤 6 个坑，直接导入上线 = 收款不发货 + 静默丢单 + 可被白嫖。** 本课给你断言级坑集复现 + 修复版工作流 + 国内可用的支付宝当面付替代方案。

## 你会学到

1. **收款发货机器人标准骨架**：IM 咨询 → 客户台账 → 付款链接 → 支付 webhook → 查表 → 原渠道交付，4 条链路 9 个功能节点
2. **6 个实锤坑全部断言复现**（docs/02-pitfalls.md）：连线接反交付逻辑整体反转 / 查无买家静默断链 / lookupValue 多余 `==` / Stripe 无验签白嫖 / phone 单字段脆弱匹配 / 双 webhook 同 path 混用
3. **修复版工作流（workflow-fixed.json）**：忠实版与修复版同一驱动脚本双跑对照，修复版 7 场景断言全绿
4. **凭证替身全家桶**：WhatsApp 企业 API → mock、Google Sheets → 本地 JSON 台账、Stripe → mock payload / 支付宝当面付
5. **支付宝当面付本地化方案**（docs/06-alipay-facepay.md）：个体工商户可签、RSA2 加签、notify_url 异步回调接 n8n webhook，0 元起步真实收款
6. **三层验证方法论**：A 单元级 27/27 → B mock 全链 15/15 → 复现脚本随课附赠可重跑

## 课程文件

| 文件 | 内容 |
|---|---|
| `workflow.json` | 官方模板原样（9 功能节点，n8n 可导入形态；含原作者凭证占位） |
| `workflow-fixed.json` | **修复版**：6 坑全修，外部依赖全替身（本地 JSON 台账 + mock 发送），可直接导入自托管 n8n 跑通 |
| `docs/00~06` | 总览 / 架构 / 坑集 / L2 验证报告 / 免费替身 / 生产加固 / 支付宝当面付本地方案 |
| `exercises/exercise.md` | 3 道动手练习（含答案） |
| `script/short-video.md` | 获客短视频口播脚本 |

## 验证结论（全部实跑，可复现）

- **A 单元级**：3 个 Extract Code 节点 + 5 个 IF 分支逻辑忠实移植，**27/27 断言全过**（2026-09-16）
- **B mock 全链（忠实版 vs 修复版双跑）**：忠实版如实复现 bug（WA 新客静默丢单、有效付款进 Fallback），修复版 7 场景全绿，**15/15 断言全过**（2026-10-25 重跑确认）
- 驱动脚本随 L2 阶段产出，投喂 payload 与断言逻辑全部开源在案，学员可一键重跑

## 一分钟看懂它干嘛

```text
客户 WhatsApp/Telegram 咨询
  → webhook 收事件 → Code 解析 → 登记客户台账 → 回发付款链接
客户付款成功
  → 支付平台 webhook → 校验有效性 → 按手机号查台账 → 识别渠道
  → WhatsApp/Telegram 原渠道发数字产品下载链接
未付款客户 → 催单链路查「已发链接未付款」→ 发提醒
无效付款（无手机号）→ Fallback 人工处理通知
```