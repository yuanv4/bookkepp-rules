---
name: bill-rule
description: 将用户粘贴的支付/账单通知内容转换为 bookkepp 项目的 notificationRules JSON 规则条目。当用户粘贴一条通知文本（无论是微信、支付宝、银行等任何来源）并请求生成规则、提取账单字段或添加新应用支持时，必须立即触发此技能。同样适用于：用户说"帮我写规则"、"识别这条通知"、"我想让 bookkepp 识别这个 App 的通知"、或粘贴任何包含金额信息的通知内容。
---

# Notification Rule Generator

你的任务是分析用户粘贴的通知文本，生成能可靠提取账单数据的 `notificationRules` 条目，格式符合 `bill_rules_default.json` 的规范。

## 规则 JSON 结构速查

```json
{
  "packageName": "com.example.app",
  "type": "EXPENSE",               // 必填：EXPENSE（支出）或 INCOME（收入）
  "titleContains": "可选：通知标题必须包含的子串",
  "contentContains": "可选：通知正文必须包含的子串",
  "fields": {
    "amount": { "regex": "正则", "template": "{1}" },         // 必填
    "counterparty": { "regex": "正则", "template": "{1}" },   // 可选
    "tradeAccount": { "regex": "正则", "template": "{1}" },   // 可选
    "tradeTime": { "regex": "正则", "template": "{1}" },      // 可选
    "category": { "regex": "正则", "template": "{1}" },       // 可选
    "detail": { "regex": "正则", "template": "{1}" },         // 可选
    "balance": { "regex": "正则", "template": "{1}" }         // 可选
  }
}
```

**字段说明**：
- `type`：必填，`EXPENSE`（支出）或 `INCOME`（收入），根据通知语义判断
- `amount`：金额，唯一必填的 fields 字段，不含货币符号（如 `123.45`）
- `counterparty`：商家/收款方名称
- `tradeAccount`：交易账号（如"余额宝"、"招商银行储蓄卡(4567)"）
- `tradeTime`：交易时间（原始字符串，系统会解析并统一为 `yyyy-MM-dd HH:mm:ss`）
- `category`：账单分类
- `detail`：备注/交易详情
- `balance`：交易后账户余额
- `regex` 和 `template` 均为必填；JSON 中正则反斜杠需双写（`\\d` 而非 `\d`）
- `template` 用 `{N}` 引用捕获组：单组提取写 `"{1}"`，多组重组如 `"{2}({1})"` 将 `尾号9359的信用卡` → `信用卡(9359)`

## 工作流程

### 第一步：收集信息

从用户提供的内容中提取：
1. **通知标题**（title）
2. **通知正文**（content / body）
3. **应用包名** — 如果用户未提供，请根据 App 名称判断（见常见包名参考），若无法确定则直接询问

如果用户只提供了通知正文（没有标题），跳过标题相关分析，继续处理。

### 第二步：分析通知结构

仔细阅读通知文本，标注出哪些片段对应哪个字段：
- **金额**：找货币符号（¥ ￥ $）、中文"元"、数字模式
- **收款方**：找"收款方"、"付款给"、商家名称、"向XX付款"等
- **交易账号**：找银行卡号（"尾号XXXX"）、支付方式名称（"余额宝"、"信用卡"）
- **时间**：找日期时间格式
- **分类/备注**：找"备注"、"说明"、"分类"等关键词

### 第三步：设计过滤条件

选择合适的 `contentContains` 或 `titleContains`（或两者）来精确匹配此类通知，避免误触发其他通知。选择最具区分性的短语（如"付款成功"比"成功"更好）。

### 第四步：编写正则表达式

每条提取器的设计原则：
- `regex` 使用捕获组 `(...)` 包裹要提取的内容
- `template` 用 `{N}` 引用对应捕获组，单组提取固定写 `"{1}"`，多组重组如 `"{2}({1})"`
- 金额：去除货币符号和千分位，只保留数字和小数点
  - 常用模式：`[¥￥]([\\d,]+\\.?\\d*)` + `template: "{1}"`
  - 或：`([\\d,]+\\.?\\d*)元` + `template: "{1}"`
- 字符串提取：使用 `(.+?)` 配合边界词，或 `(.+)` 到行尾
- 时间：匹配常见格式如 `(\\d{4}[-/]\\d{2}[-/]\\d{2}[\\s T]\\d{2}:\\d{2}:\\d{2})`
- 测试：在脑中用原文跑一遍正则，确认能命中

### 第五步：输出规则

输出完整的 JSON 代码块，并在其下方列出：
1. 各字段的匹配验证（"字段X 从 '原文片段' 中提取出 '值'"）
2. 如有字段无法从通知中提取，说明原因
3. 如包名不确定，给出最可能的候选并说明如何查找

## 常见应用包名参考

| 应用 | 包名 |
|------|------|
| 支付宝 | `com.eg.android.AlipayGphone` |
| 微信 | `com.tencent.mm` |
| 工商银行 | `com.icbc` |
| 建设银行 | `com.chinamobile.mcloud` / `com.ccb.e3handy` |
| 招商银行 | `com.cmbchina.ccd.pluto.cmbActivity` |
| 农业银行 | `com.android.bankabc` |
| 中国银行 | `com.android.bank` |
| 京东 | `com.jingdong.app.mall` |
| 美团 | `com.sankuai.meituan` |
| 滴滴 | `com.sdu.didi.psnger` |

## 示例

### 示例 1：支付宝交易提醒（仅提取金额）

**输入通知**：
- 应用：`com.eg.android.AlipayGphone`
- 标题：`交易提醒`
- 内容：`你有一笔6.00元的支出，点击领取10个支付宝积分。`

**输出**：
```json
{
  "packageName": "com.eg.android.AlipayGphone",
  "type": "EXPENSE",
  "titleContains": "交易提醒",
  "fields": {
    "amount": {
      "regex": "一笔([\\d,]+\\.?\\d*)元",
      "template": "{1}"
    }
  }
}
```

验证：
- `amount`：从 `一笔6.00元` 提取 → `6.00` ✓
- `counterparty` / `tradeAccount` / `tradeTime`：通知正文无相关信息，跳过

---

### 示例 2：建设银行动账提醒（多组 template + 非标准时间格式）

**输入通知**：
- 应用：`com.chinamworld.main`
- 标题：`动账提醒`
- 内容：`您尾号9359的信用卡账户3月25日8时52分消费人民币6.00元。点击查看>>`

**输出**：
```json
{
  "packageName": "com.chinamworld.main",
  "type": "EXPENSE",
  "titleContains": "动账提醒",
  "contentContains": "消费人民币",
  "fields": {
    "amount": {
      "regex": "消费人民币([\\d,]+\\.?\\d*)元",
      "template": "{1}"
    },
    "tradeAccount": {
      "regex": "尾号(\\d+)的(信用卡|储蓄卡)",
      "template": "{2}({1})"
    },
    "tradeTime": {
      "regex": "(\\d+月\\d+日\\d+时\\d+分)",
      "template": "{1}",
      "inputFormat": "M月d日H时m分",
      "outputFormat": "yyyy-MM-dd HH:mm:ss"
    }
  }
}
```

验证：
- `amount`：从 `消费人民币6.00元` 提取 → `6.00` ✓
- `tradeAccount`：从 `尾号9359的信用卡` 提取，`{2}({1})` 重组 → `信用卡(9359)` ✓
- `tradeTime`：从 `3月25日8时52分` 提取，按 `M月d日H时m分` 解析后补全年份 → `2026-03-25 08:52:00` ✓
- `contentContains` 设为 `消费人民币`，避免与同 App 其他动账通知（如转账、存款）混淆

---

## 注意事项

- 如果一条通知同时有收款和付款，建议通过 `contentContains` 区分，分别生成两条规则
- 金额中的千分位逗号（如 `1,234.56`）保留在提取结果中，应用会自动处理
- 通知内容中如有换行，正则中用 `[\\s\\S]*?` 或 `.` 配合 DOTALL 标志（提示用户如有需要）
- 生成的规则条目直接可粘贴进 `bill_rules_default.json` 的 `notificationRules` 数组
