# 06 · 支付宝当面付本地方案（Stripe 的中国替身）

Stripe 在国内收款受限，而**支付宝当面付个体工商户即可签约、0 元开通、支持异步回调**——是把本课骨架真正落地国内的关键一环。本方案已在本机完成真实商户入驻与应用配置验证（2026-09-22），以下全流程可复现。

## 方案架构

```text
客户咨询（TG/企微/飞书）→ n8n 登记台账 → Code 节点调当面付接口
  → 生成扫码付二维码（alipay.trade.precreate 预下单）→ 发给客户
客户扫码付款 → 支付宝 POST 异步通知到 notify_url（n8n webhook）
  → RSA2 验签 + 验金额/订单号 → 查台账匹配客户 → 原渠道发货 → 回 success
```

## 开通四步（个体工商户亲测可行）

1. **商家入驻**：支付宝商家平台（b.alipay.com）注册个体工商户商户，开通余额账户
2. **创建应用**：登录 open.alipay.com → 创建「网页应用」→ 拿到 **APPID**（形如 `2021xxxxxxxxxxxx`）
3. **签约当面付**：应用详情里申请「当面付」产品，提交审核（实测约 24 小时内生效）
4. **配置密钥（RSA2）**：
   - 本地生成 RSA2 密钥对（2048 位）：`openssl genrsa -out alipay-app-private.pem 2048` 再导出公钥
   - 开放平台「接口加签」选**密钥模式**，粘贴**应用公钥**单行文本
   - 保存后回拿**支付宝公钥**——它用于验签回调，和你的应用私钥是两条线的两把钥匙

## n8n 侧接入要点

- **预下单**：Code 节点调 `alipay.trade.precreate`，参数 `out_trade_no`（=台账订单号，幂等键）、`total_amount`、`subject`；签名 = 按支付宝规范拼接待签名字符串 → SHA256withRSA（应用私钥）→ base64。返回里的 `qr_code` 就是给客户的扫码链接
- **回调接收**：webhook 节点挂 `notify_url`（需公网可达，本机调试可用内网穿透）；Code 节点用**支付宝公钥**验签，验 `out_trade_no` 存在且金额一致、`trade_status` ∈ {TRADE_SUCCESS, TRADE_FINISHED}，全部通过才进入交付链，最后**必须回纯文本 `success`**（否则支付宝按策略重发 8 次）
- **交付匹配**：`out_trade_no` 直接编码客户标识（如 `TG88001-20261025-001`），彻底摆脱 phone 单字段匹配（坑#5 的终极解法）

## 与 Stripe 链路的对照

| 维度 | Stripe（模板原样） | 支付宝当面付（本方案） |
|---|---|---|
| 收款主体 | 需海外主体 | 个体工商户即可 |
| 发起支付 | checkout 付款链接 | 预下单生成扫码二维码 |
| 回调事件 | checkout.session.completed | trade_status=TRADE_SUCCESS |
| 验签 | endpointSecret HMAC | 支付宝公钥 RSA2 |
| 匹配键 | customer_details.phone（脆弱） | out_trade_no（自掌控，可靠） |

## ⚠️ 密钥红线

应用私钥、支付宝公钥、商户号、APPID 均为敏感信息：**只存 n8n 凭证/环境变量，绝不写进工作流 JSON，绝不进 Git 仓库**。本课程包中所有相关字段一律使用占位符（`<YOUR_APPID>`、`<YOUR_APP_PRIVATE_KEY>`、`<ALIPAY_PLATFORM_PUBLIC_KEY>`），请按上表自行开通后填入。