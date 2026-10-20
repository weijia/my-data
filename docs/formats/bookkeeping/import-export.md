# Bookkeeping 导入 / 导出格式

本文档描述 Bookkeeping 应用的 JSON 导入与导出格式、自动识别规则，以及供 AI 生成记录的提示词模板。

**源码参考**：`src/components/apps/BookkeepingApp.tsx` 中的 `parseImport()`、`detectRowKind()`、`normalizeRow()`、`normalizePriceRow()`、`AI_PROMPT_TEMPLATE`

## 支持顶层格式

`parseImport()` 支持以下几种顶层结构：

| 形式 | 示例 | 说明 |
|------|------|------|
| 纯数组 | `[ {...}, {...} ]` | 每条按字段自动识别 tx / price |
| 交易包装 | `{ "transactions": [ ... ] }` | 仅按交易记录解析 |
| 价格包装 | `{ "prices": [ ... ] }` | 仅按价格记录解析 |
| 混合包装 | `{ "transactions": [...], "prices": [...] }` | 两个数组分别解析 |
| 通用数组包装 | `{ "records": [ ... ] }` | 取首个数组字段，逐条自动识别 |

同时存在 `transactions` 与 `prices` 时各自解析；只有其中一个时按对应类型解析。

## 逐条自动识别规则

对纯数组 / 通用包装中的每一条记录，通过 `detectRowKind()` 判断类型：

| 判定顺序 | 条件 | 识别为 |
|----------|------|--------|
| 1 | 含 `kind` 字段 | **价格（price）** |
| 2 | 含 `amount` 且（含 `type` 或含 `category`） | **交易（tx）** |
| 3 | 含 `unitPrice` 且不含 `type` | **价格（price）** |
| — | 都不满足 | 跳过并计入无效数 |

> 因此同一个数组里可以混排交易与价格记录，导入时会自动分流到对应的数据集合。

## 导入流程与反馈

1. 触发导入（三种方式）：
   - 点工具栏「导入」按钮 → 优先读系统剪贴板，无 JSON 时回退到文件选择
   - 非输入焦点处 `Ctrl+V` / `Cmd+V` → 直接解析粘贴内容
   - 点工具栏文件图标 → 选择本地 `.json` / `.txt` 文件
2. 解析成功 → 弹出「导入预览」，显示识别到的记录数 / 价格数与跳过数
3. 选择导入模式：
   - **合并**：新记录追加到现有数据
   - **替换**：用新数据替换对应集合
4. 确认导入 → 写入并同步

> 解析失败、剪贴板无有效 JSON、无有效记录等情况均通过顶部 Toast 反馈。

## 导出格式

导出按当前 Tab 输出对应的 JSON 数组（缩进 2 空格），文件名为：

- 记录 Tab：`bookkeeping-tx-{YYYY-MM-DD}.json`
- 价格 Tab：`bookkeeping-price-{YYYY-MM-DD}.json`

字段格式见：
- [交易记录导出格式](./transactions.md#导出格式)
- [价格记录导出格式](./prices.md#导出格式)

> 导出的 JSON 可直接作为模板交给 AI，或再次导入。

## AI 提示词模板

应用内置统一提示词，点击 `?` 按钮即可复制。将其粘贴到任意 AI（ChatGPT / Claude / 豆包 等），把占位符替换为你的描述，AI 会输出可直接导入的混合 JSON：

````text
你是一个智能记账助手。请理解以下内容（可能是消费流水、也可能是商品价格、可能同时有两者），
输出一个严格的 JSON 数组，不要输出任何其他文字（不要解释、不要 markdown 代码块标记）。

内容：{在这里粘贴你的流水或价格描述，例如：今天午餐麻辣烫35，地铁6；鸡蛋5.8一斤；超市券75买100}

请判断每条内容属于哪种类型，用不同字段输出：

【A. 记账记录（支出/收入）】
特征：有金额支出或收入，带 type/category/amount 字段
字段：date(YYYY-MM-DD)、type("expense"支出/"income"收入)、category(分类)、amount(金额)、note(备注，可选)、product(商品名，可选)
分类建议：餐饮/交通/购物/居家/娱乐/医疗/工资/理财/其他

【B. 价格记录（报价/单价/套餐/代金券）】
特征：描述"X 多少钱一斤"、套餐价格、代金券面值等，用 kind 字段区分
- kind:"item" 单品：product、unit、unitPrice、date
- kind:"combo" 套餐：name、unitPrice、items[] (product/unit/quantity)、date
- kind:"voucher" 代金券：name、cost(实付)、faceValue(面值)、date

数组里可以混合两类，系统会自动识别。

示例输出：
[
  {"date":"2026-10-20","type":"expense","category":"餐饮","amount":35,"note":"午餐","product":"麻辣烫"},
  {"date":"2026-10-20","type":"expense","category":"交通","amount":6,"note":"地铁"},
  {"kind":"item","date":"2026-10-20","product":"鸡蛋","unit":"斤","unitPrice":5.8},
  {"kind":"voucher","date":"2026-10-20","name":"超市券","cost":75,"faceValue":100}
]
````

## 完整导入示例

一段混合了交易与价格、且被自动分流的 JSON：

```json
[
  { "date": "2026-10-20", "type": "expense", "category": "餐饮", "amount": 35, "note": "午餐" },
  { "date": "2026-10-20", "type": "expense", "category": "交通", "amount": 6, "note": "地铁" },
  { "kind": "item", "date": "2026-10-20", "product": "鸡蛋", "unit": "斤", "unitPrice": 5.8 },
  { "kind": "voucher", "date": "2026-10-20", "name": "超市券", "cost": 75, "faceValue": 100 }
]
```

导入预览将显示：**2 条记录、2 条价格**，分别写入 `transactions` 与 `prices`。

[返回索引](../bookkeeping-format.md)
