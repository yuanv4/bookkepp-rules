# bill_rules_default.json — 账单规则配置指南

此文件定义了无障碍服务提取账单信息的完整规则集，包含窗口规则和通知规则。
规则通过**订阅模式**分发：应用启动时从配置的远端 URL 拉取（支持 ETag 缓存），不再内置于 APK。
本文件托管于 `rule/` 目录，可通过 raw.githubusercontent.com 直接订阅。

---

## 根结构

```jsonc
{
  "name": "规则集名称（订阅页面展示）",
  "author": "作者（订阅页面展示为 @author）",
  "version": 1,
  "windowRules": [ /* 窗口规则数组 */ ],
  "notificationRules": [ /* 通知规则数组 */ ]
}
```

---

## 窗口规则（windowRules）

从应用窗口（账单详情页）提取账单信息，基于**标签文本定位值节点**。

```jsonc
{
  "packageName": "com.eg.android.AlipayGphone",
  "type": "EXPENSE",              // 必填：收支类型，固定值 "EXPENSE"（支出）或 "INCOME"（收入）
  "fields": {
    "amount": {
      "交易成功": {
        "valueDirection": "sibling:prev",  // 值节点相对于标签的树结构位置（见下方说明）
        "valueRegex": "([\\d.]+)",         // 可选：从节点文本中正则提取值
        "valueGroup": 1                    // 可选：正则捕获组序号，默认 0（整个匹配）
      },
      "自动扣款成功": {
        "valueDirection": "sibling:prev"
      }
    },
    "counterparty": {
      "收款方全称": { "valueDirection": "sibling:next" }
    },
    "tradeAccount": {
      "付款方式": { "valueDirection": "sibling:next" }
    },
    "tradeTime": {
      "支付时间": {
        "valueDirection": "sibling:next",
        "inputFormat": "yyyy年M月d日 HH:mm",   // 可选：UI 显示的时间格式（见下方说明）
        "outputFormat": "yyyy-MM-dd HH:mm:ss"  // 可选：统一输出格式，默认 yyyy-MM-dd HH:mm:ss
      }
    },
    "category": {
      "账单分类": { "valueDirection": "sibling:next" }
    },
    "detail": {
      "转账备注": { "valueDirection": "sibling:next" },
      "付款备注": { "valueDirection": "sibling:next" },
      "交易详情": { "valueDirection": "sibling:next" }
    },
    "balance": {
      "账户余额": { "valueDirection": "sibling:next" }
    }
  }
}
```

### fields 格式说明

每个字段名（`amount`、`counterparty` 等）对应一个 **label-as-key Map**：
- **key**：标签文本，即界面上紧邻值的说明文字。
- **value**：定位器对象，包含：
  - `valueDirection`（必填）：值节点相对于标签的**树结构位置**，支持以下值：
    - `"self"` — 标签节点本身（必须配合 `valueRegex` 提取子字符串）
    - `"parent"` — 直接父节点
    - `"child:N"` — 第 N 个子节点（N 从 0 起，按现存顺序）
    - `"sibling:next"` — 下一个兄弟节点（等价于 `sibling:next:1`）
    - `"sibling:prev"` — 上一个兄弟节点（等价于 `sibling:prev:1`）
    - `"sibling:next:N"` — 向后第 N 个兄弟（N ≥ 1，按现存顺序）
    - `"sibling:prev:N"` — 向前第 N 个兄弟（N ≥ 1，按现存顺序）
    > 所有树方向均基于 UI 可访问性树的**现存节点顺序**，由调试拾取器自动推断生成。
  - `valueRegex`（可选）：从值节点文本中正则提取所需内容。
  - `valueGroup`（可选）：正则捕获组序号，默认 `0`。
  - `inputFormat`（可选，仅 `tradeTime`）：UI 显示的时间格式，使用 `SimpleDateFormat` pattern（如 `"yyyy年M月d日 HH:mm"`）。
  - `outputFormat`（可选，仅 `tradeTime`）：输出时间格式，默认 `"yyyy-MM-dd HH:mm:ss"`。配置 `inputFormat` 后引擎自动解析并重格式化；若 `inputFormat` 不含年份，自动从时间戳补全。

同一字段可配置多个标签（如 `amount` 同时匹配"交易成功"和"自动扣款成功"），引擎按顺序尝试，取第一个命中的值。

---

## 通知规则（notificationRules）

从通知栏消息中提取账单信息，基于**正则表达式**匹配通知内容。

```jsonc
{
  "packageName": "com.eg.android.AlipayGphone",
  "type": "EXPENSE",              // 必填：收支类型，固定值 "EXPENSE"（支出）或 "INCOME"（收入）
  "contentContains": "付款成功",   // 通知内容必须包含此字符串才触发（可选）
  "fields": {
    "amount": {
      "regex": "[¥￥]([\\d,]+\\.?\\d*)",
      "group": 1
    },
    "counterparty": {
      "regex": "收款方[:：]\\s*(.+)",
      "group": 1
    },
    "tradeAccount": {
      "regex": "付款方式[:：]\\s*(.+)",
      "group": 1
    },
    "tradeTime": {
      "regex": "(\\d{4}-\\d{2}-\\d{2} \\d{2}:\\d{2}:\\d{2})",
      "group": 1
    },
    "category": {
      "regex": "账单分类[:：]\\s*(.+)",
      "group": 1
    },
    "detail": {
      "regex": "备注[:：]\\s*(.+)",
      "group": 1
    },
    "balance": {
      "regex": "余额([\\d,]+\\.\\d{2})元",
      "template": "{1}"
    }
  }
}
```

### fields 格式说明

每个字段名对应一个提取器对象：
- `regex`：用于从通知文本中提取该字段值的正则表达式（必填）。
- `template`：输出模板（必填），用 `{N}` 引用捕获组（N 从 0 起，0 为完整匹配）。单组提取用 `"{1}"`，多组重组如 `"{2}({1})"` 将 `尾号9359的信用卡` 转换为 `信用卡(9359)`。
- `inputFormat`：（可选，仅 `tradeTime`）提取值的时间格式，使用 `SimpleDateFormat` pattern（如 `"M月d日H时m分"`）。
- `outputFormat`：（可选，仅 `tradeTime`）输出时间格式，默认 `"yyyy-MM-dd HH:mm:ss"`。配置 `inputFormat` 后引擎自动解析并重格式化；若 `inputFormat` 不含年份，自动从通知 timestamp 补全。

`amount` 是唯一必填字段；其余字段缺失时由系统按 `timestamp` 等兜底填充。

---

## 注意事项

- **冷却时间**：窗口规则触发间隔固定为项目默认值（5000ms），通知规则每条通知仅触发一次。
- **去重**：系统自动生成 `orderId`（格式 `auto_<packageName>_<amount>_<tradeTime>`），按唯一索引去重。
- **时间格式**：`tradeTime` 始终以 `yyyy-MM-dd HH:mm:ss` 格式存储；两套引擎均支持以通知/事件时间戳兜底，提取到实际值时不覆盖。
- **订阅刷新**：应用通过 ETag 缓存避免重复下载；可在设置页手动触发刷新。
