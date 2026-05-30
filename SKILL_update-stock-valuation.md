---
name: update-stock-valuation
description: 更新腾讯文档中美股核心股票前瞻估值表的实时股价。用户提到更新估值表、更新股价、前瞻估值、估值表、valuation sheet、腾讯文档 Sheet 股价、2026年4月/7月 Sheet 时自动使用。
---

# 美股估值表实时股价更新

用于把腾讯文档估值表中指定 Sheet 的股票现价批量更新到 A 列价格行。默认行情来源为富途 OpenAPI；富途不支持的 OTC/ADR 标的使用权威公开网站补齐。

## 输入

需要从用户请求中识别：
1. 腾讯文档 Sheet URL，例如 `https://docs.qq.com/sheet/DWnVWZnFTUGtGWXFO?tab=gkrzwc`
2. 目标子表名，例如 `2026年4月`、`2026年7月`

如果 URL 中的 `tab` 与用户指定的 Sheet 名称不一致，以用户指定的 Sheet 名称为准。

## 表格结构

- 第 0 行：主表头
- 第 1 行：子表头
- 第 2 行起：每只股票固定占 5 行
  - 偏移 `+0`：A 列为股票代码
  - 偏移 `+1`：A 列为实时股价，更新目标
  - 偏移 `+2..+4`：历史 EPS 等数据
- 遇到 `MSTR` 且价格行是 `无法提供估值` 时，保留原值，不覆盖

## 工具约定

- 腾讯文档读写使用 `tencent-sheetengine`
  - `get_sheet_info`
  - `get_cell_data`
  - `set_range_value`
- 股票快照优先使用富途脚本：
  - 首选路径：`skills/futuapi/scripts/quote/get_snapshot.py`
  - 若不存在，使用 `/Users/debyecao/.cursor/skills/futuapi/scripts/quote/get_snapshot.py`
- 读取 A 列使用 `return_csv: true`
- 批量写入使用 `set_range_value`，每批最多 60 个单元格

## 更新流程

### 1. 定位目标 Sheet

1. 调用 `get_sheet_info` 读取所有子表
2. 按 `sheet_name` 精确匹配用户指定名称
3. 记录 `sheet_id` 和 `row_count`
4. 找不到时列出可用子表名并停止

### 2. 解析股票与价格行

读取目标 Sheet 的 A 列，按 5 行一组扫描：

```python
for row in range(2, row_count, 5):
    code = rows[row].strip()
    price_row = row + 1
```

只接受形如 `NVDA`、`BRK.B`、`TCEHY` 的 ticker。构造：

```python
positions = {"NVDA": 3, "AAPL": 508, "TCEHY": 1158}
```

跳过规则：
- 空代码跳过
- `MSTR` 且价格行为 `无法提供估值` 时跳过并在结果中说明

### 3. 富途批量取价

把美股 ticker 转为 `US.{ticker}`，例如：

```bash
python3 /Users/debyecao/.cursor/skills/futuapi/scripts/quote/get_snapshot.py US.NVDA US.AAPL US.BRK.B --json
```

解析输出中每个对象的 `last_price`。如果整批失败，用二分方式拆批定位失败 ticker，避免一个 OTC 标的拖垮整批。

### 4. OTC/ADR 权威补价

富途通常拿不到 OTC 标的，例如 `TCEHY`。这类标的必须使用权威公开来源补齐，不能使用用户截图、手动上次值或港股折算作为默认值。

优先级：
1. Yahoo Finance 美国主站：`https://finance.yahoo.com/quote/{ticker}/`
2. OTC Markets 官方页：`https://www.otcmarkets.com/stock/{ticker}/quote`
3. Google Finance 或 Morningstar 仅作交叉验证

对 `TCEHY` 的硬规则：
- 必须搜索 `TCEHY Yahoo Finance current price {当前日期年份} Tencent Holdings ADR latest quote`
- 选择带有明确日期/时间且最新的 Yahoo Finance `finance.yahoo.com/quote/TCEHY/` 报价
- 如果搜索结果互相冲突，优先 `finance.yahoo.com/quote/TCEHY/`，其次 OTC Markets
- 不要用 `sg.finance.yahoo.com`、`uk.finance.yahoo.com` 等页面中的过旧片段覆盖美国主站更近报价
- 不要用 `HK.00700 / USDHKD` 折算，除非用户明确要求折算
- 不要把用户截图或用户手动修正值当成自动 fallback 来源
- 汇报时说明 `TCEHY` 来源 URL 和报价时间

如果权威来源也没有明确报价，停止并说明缺失，不要猜价格。

### 5. 批量写入

构造 `set_range_value` 参数：

```json
{
  "file_url": "<docs.qq.com URL>",
  "sheet_id": "<sheet_id>",
  "values": [
    {"row": 3, "col": 0, "value_type": "NUMBER", "number_value": 211.14}
  ]
}
```

每 60 个单元格一批写入。写入前必须确保所有可更新 ticker 都有价格；有缺失则先处理缺失，不要部分写入后才发现。

### 6. 验证

写入后重新读取目标 Sheet A 列，验证：
- 可更新价格行数量等于写入数量
- 所有可更新价格行都是数字
- `MSTR` 仍保留 `无法提供估值`
- 抽查至少：`NVDA`、`AAPL`、`TCEHY`、`ZS`

验证输出示例：

```json
{
  "updatable_rows": 248,
  "numeric_price_rows": 248,
  "non_numeric_after_update": [],
  "skipped_preserved": ["MSTR"],
  "samples": {
    "NVDA": "211.14",
    "AAPL": "312.06",
    "TCEHY": "54.62",
    "ZS": "139.73"
  }
}
```

## 汇报格式

最终回复用中文，简洁说明：
- 已更新的 Sheet 名称
- 写入股票数
- 富途返回数量
- fallback 数量及具体 ticker、来源链接
- 跳过项，例如 `MSTR`
- 验证结论和抽查值

示例：

```markdown
已更新 `2026年7月` Sheet 的股价。

本次写入 `248` 个可更新价格：`247` 个来自富途快照，`TCEHY` 富途未返回，按 Yahoo Finance 最新明确报价写入 `54.62`（5/28 收盘，来源：[Yahoo Finance](https://finance.yahoo.com/quote/TCEHY/)）。`MSTR` 保留“无法提供估值”，未覆盖。

已重新读取验证：`248/248` 个可更新价格行均为数字。抽查：`NVDA=211.14`、`AAPL=312.06`、`TCEHY=54.62`、`ZS=139.73`。
```

## 常见错误

- 错误：使用 URL `tab` 对应的子表，而不是用户指定的 Sheet 名称  
  正确：始终按 `sheet_name` 匹配用户指定 Sheet
- 错误：`TCEHY` 使用截图、旧值或港股折算  
  正确：使用 Yahoo Finance/OTC Markets 明确报价，并汇报来源
- 错误：把 `参考BTC即可` 等说明误判为 ticker  
  正确：按 5 行块结构扫描，只看每块第 1 行
- 错误：写完不验证  
  正确：写入后必须重新读取并检查数字行数量
