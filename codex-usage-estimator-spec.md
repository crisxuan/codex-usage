# Codex 订阅用量估算器 Spec

版本：v0.1  
状态：Draft  
日期：2026-05-10  
作者：Codex

## 1. 背景

Codex 当前在订阅场景中主要展示使用额度或剩余额度，但不会直接展示每次任务消耗的真实 token 数。用户希望自己构建一个本地工具，通过任务请求数量、可见上下文 token 估算、工具调用量和订阅用量变化，分析自己的 Codex 使用模式。

这个工具不用于计算 OpenAI API 的 token 账单，也不声称能还原 Codex 内部真实 token。它的目标是建立一个个人化的、可校准的订阅额度消耗估算系统。

## 2. 目标

构建一个本地 CLI 工具，用于按任务组记录和分析 Codex 使用情况。

核心问题：

- 一类 Codex 任务大概有多“贵”？
- 一次问答、小改动、重构、长时间 agent 分别消耗多少订阅额度？
- 请求数量、可见 token、工具调用数量和订阅额度消耗之间有什么关系？
- 不同模型、不同执行模式、本地或云端任务的消耗差异如何？

MVP 输出指标：

- 任务组数量
- turn 数量
- 估算请求数
- 估算可见 token 数
- 工具调用次数
- 订阅用量前后变化
- 每 1% 订阅额度对应的估算可见 token
- 按任务组、模型、执行模式聚合的消耗报告

## 3. 非目标

本工具不做以下事情：

- 不计算 OpenAI API 真实 token 账单。
- 不声明能获取 Codex 内部真实 token。
- 不假设 1 条用户消息等于 1 次模型请求。
- 不假设 1 次模型请求等于固定 token。
- 不假设订阅额度百分比可以直接映射到 API token 价格。
- MVP 不自动读取 Codex UI 的用量条。
- MVP 不做图形界面。

所有 token 相关字段和报告文案必须显式标注为 estimated。

## 4. 核心概念

### 4.1 Task Group

Task Group 是分析的基本归属单位。一个任务组可以对应一次重构、一次调研、一个 bug 修复、一段连续会话，或用户手动定义的任意工作单元。

示例：

- `quick-chat`
- `repo-refactor`
- `bugfix-login`
- `long-agent-run`
- `research-openai-docs`

### 4.2 Usage Snapshot

Usage Snapshot 是某个时间点的订阅用量记录。MVP 由用户手动输入。

示例：

- 任务开始前：`usage = 42`
- 任务结束后：`usage = 47`
- 则本任务组区间内的订阅额度变化为 `5%`

### 4.3 Turn Record

Turn Record 表示一次可记录的 Codex 交互片段。它不要求严格对应内部模型请求，只记录用户可观察或可导入的文本与行为信息。

Turn 可以来自：

- 手动粘贴的 transcript
- 导入的 Markdown 文件
- 后续自动解析的日志
- 后续从 Codex 会话导出的结构化数据

### 4.4 Visible Tokens

Visible Tokens 指用户本地可见文本的 token 估算值，包括：

- 用户输入
- 助手输出
- 工具输出
- 文件上下文
- transcript 中可见的代码、diff、终端输出

它不包括不可见系统提示、隐藏工具 schema、模型内部 reasoning tokens、压缩历史等内容。

## 5. 数据模型

MVP 使用 JSONL 存储。字段命名使用 camelCase。所有估算 token 字段必须包含 `Estimated`。

### 5.1 TaskGroup

```ts
type TaskGroup = {
  id: string
  name: string
  createdAt: string
  description?: string
  labels?: string[]
}
```

示例：

```json
{"id":"tg_01","name":"repo-refactor","createdAt":"2026-05-10T12:00:00+08:00","labels":["code","refactor"]}
```

### 5.2 UsageSnapshot

```ts
type UsageSnapshot = {
  id: string
  taskGroupId: string
  timestamp: string
  usagePercent?: number
  remainingPercent?: number
  source: "manual" | "screenshot" | "dom" | "log"
  note?: string
}
```

规则：

- `usagePercent` 表示已使用百分比。
- `remainingPercent` 表示剩余百分比。
- 二者可以只填一个。
- 如果同时提供二者，二者应满足 `usagePercent + remainingPercent = 100`，允许 0.1 的浮点误差。

示例：

```json
{"id":"us_01","taskGroupId":"tg_01","timestamp":"2026-05-10T12:01:00+08:00","usagePercent":42,"source":"manual"}
```

### 5.3 TurnRecord

```ts
type TurnRecord = {
  id: string
  taskGroupId: string
  timestamp: string

  model?: string
  mode?: "local" | "cloud" | "unknown"
  taskType?: "simple_chat" | "small_code_change" | "medium_code_task" | "large_repo_task" | "long_running_agent" | "unknown"

  source?: "manual" | "file" | "log"
  sourcePath?: string

  userText?: string
  assistantText?: string
  toolText?: string
  fileContextText?: string

  userTokensEstimated: number
  assistantTokensEstimated: number
  toolTokensEstimated: number
  fileContextTokensEstimated: number
  visibleTokensEstimated: number
  effectiveTokensEstimated?: number

  requestCountEstimated: number
  toolCallCount: number
}
```

示例：

```json
{"id":"tr_01","taskGroupId":"tg_01","timestamp":"2026-05-10T12:10:00+08:00","model":"codex-max","mode":"local","taskType":"medium_code_task","source":"file","sourcePath":"transcript.md","userTokensEstimated":1200,"assistantTokensEstimated":3400,"toolTokensEstimated":6000,"fileContextTokensEstimated":2500,"visibleTokensEstimated":13100,"effectiveTokensEstimated":52400,"requestCountEstimated":1,"toolCallCount":6}
```

## 6. 存储布局

默认存储目录：

```text
.codex-usage/
  groups.jsonl
  snapshots.jsonl
  turns.jsonl
  config.json
```

### 6.1 config.json

```json
{
  "version": 1,
  "defaultEncoding": "o200k_base",
  "defaultLanguage": "auto",
  "defaultModel": "unknown",
  "defaultMode": "unknown",
  "multipliers": {
    "simple_chat": 1.5,
    "small_code_change": 2.5,
    "medium_code_task": 4,
    "large_repo_task": 6,
    "long_running_agent": 8,
    "unknown": 3
  }
}
```

## 7. Token 估算

### 7.1 Tokenizer

MVP 使用 Python + `tiktoken`。

模型未知时使用：

```text
o200k_base
```

估算函数：

```python
import tiktoken

def count_tokens(text: str, model: str | None = None) -> int:
    if not text:
        return 0

    if model:
        try:
            encoding = tiktoken.encoding_for_model(model)
        except KeyError:
            encoding = tiktoken.get_encoding("o200k_base")
    else:
        encoding = tiktoken.get_encoding("o200k_base")

    return len(encoding.encode(text))
```

### 7.2 可见 token 公式

```text
visibleTokensEstimated =
  userTokensEstimated
+ assistantTokensEstimated
+ toolTokensEstimated
+ fileContextTokensEstimated
```

### 7.3 有效 token 公式

有效 token 是为了粗略补偿不可见上下文、历史压缩、系统提示、工具 schema、重复上下文等不可观察因素。

```text
effectiveTokensEstimated =
  visibleTokensEstimated * multiplier
```

默认 multiplier：

```text
simple_chat:        1.5
small_code_change:  2.5
medium_code_task:   4
large_repo_task:    6
long_running_agent: 8
unknown:            3
```

注意：`effectiveTokensEstimated` 不是 OpenAI 真实 token，只是本工具的校准指标。

## 8. 订阅额度计算

### 8.1 usage delta

当使用已用百分比时：

```text
usageDeltaPercent = after.usagePercent - before.usagePercent
```

当使用剩余百分比时：

```text
usageDeltaPercent = before.remainingPercent - after.remainingPercent
```

若结果小于 0，应提示用户快照顺序可能错误。

### 8.2 每 1% 额度对应 token

```text
visibleTokensPerUsagePercent =
  sum(visibleTokensEstimated) / usageDeltaPercent
```

```text
effectiveTokensPerUsagePercent =
  sum(effectiveTokensEstimated) / usageDeltaPercent
```

### 8.3 每请求 token

```text
visibleTokensPerRequest =
  sum(visibleTokensEstimated) / sum(requestCountEstimated)
```

### 8.4 每工具调用 token

```text
visibleTokensPerToolCall =
  sum(visibleTokensEstimated) / sum(toolCallCount)
```

## 9. CLI 设计

命令名：

```bash
codex-usage
```

### 9.1 初始化

```bash
codex-usage init
```

行为：

- 创建 `.codex-usage/`
- 创建 `groups.jsonl`
- 创建 `snapshots.jsonl`
- 创建 `turns.jsonl`
- 创建默认 `config.json`

### 9.2 创建任务组

```bash
codex-usage group create "repo-refactor"
```

可选参数：

```bash
--description "Large repo refactor task"
--label code
--label refactor
```

### 9.3 查看任务组

```bash
codex-usage group list
```

输出：

```text
ID      Name           Labels          Created
tg_01   repo-refactor  code,refactor   2026-05-10 12:00
```

### 9.4 记录用量快照

```bash
codex-usage snapshot --group repo-refactor --usage 42
```

或：

```bash
codex-usage snapshot --group repo-refactor --remaining 58
```

可选参数：

```bash
--note "before starting refactor"
--source manual
```

### 9.5 添加 turn

从文件导入：

```bash
codex-usage turn add \
  --group repo-refactor \
  --file transcript.md \
  --model codex-max \
  --mode local \
  --task-type medium_code_task
```

从标准输入导入：

```bash
cat transcript.md | codex-usage turn add --group repo-refactor --stdin
```

手动指定请求数和工具调用数：

```bash
codex-usage turn add \
  --group repo-refactor \
  --file transcript.md \
  --requests 3 \
  --tool-calls 8
```

MVP 中文本分段规则可以先简单处理：

- 全部导入文本先计入 `assistantText` 或 `toolText` 之外的 generic visible text。
- 若 transcript 中存在显式标记，后续再解析到 user、assistant、tool。

推荐 transcript 标记格式：

```markdown
<!-- codex-usage:user -->
用户文本

<!-- codex-usage:assistant -->
助手文本

<!-- codex-usage:tool -->
工具输出

<!-- codex-usage:file-context -->
文件上下文
```

### 9.6 生成报告

全部报告：

```bash
codex-usage report
```

按任务组：

```bash
codex-usage report --group repo-refactor
```

按时间：

```bash
codex-usage report --since 7d
codex-usage report --from 2026-05-01 --to 2026-05-10
```

按模型：

```bash
codex-usage report --model codex-max
```

按执行模式：

```bash
codex-usage report --mode local
```

示例输出：

```text
Estimated Codex Usage Report

Task Group      Turns  Requests  Tool Calls  Visible Tokens  Effective Tokens  Usage Δ  Visible / 1%
repo-refactor   12     18        64          184,200         736,800           5.0%     36,840
quick-fix       8      8         19          21,900          54,750            0.8%     27,375
research        5      11        22          96,500          289,500           3.2%     30,156

Notes:
- Token values are estimated from locally visible text.
- Usage percentage is manually captured subscription usage.
- Effective tokens are heuristic, not OpenAI billing tokens.
```

### 9.7 导出

```bash
codex-usage export --format csv --output usage.csv
codex-usage export --format json --output usage.json
```

MVP 可以只支持 CSV。

## 10. Transcript 解析策略

### 10.1 MVP 策略

MVP 优先保证可用，不追求完美解析。

处理顺序：

1. 如果存在 `codex-usage` HTML 注释标记，则按标记分段。
2. 如果没有标记，则将全文计入 `assistantText` 或 `toolText` 之外的 generic text，并最终纳入 `visibleTokensEstimated`。
3. 如果用户指定 `--kind user|assistant|tool|file-context`，则全文计入指定类别。

CLI 参数：

```bash
--kind user
--kind assistant
--kind tool
--kind file-context
--kind mixed
```

默认：

```text
mixed
```

### 10.2 后续策略

后续可以支持自动识别：

- `user` / `assistant` 对话头
- shell 输出块
- diff 块
- Markdown code fences
- tool call 元数据
- Codex 导出的结构化日志

## 11. 校准策略

工具的价值来自持续记录后的个人校准。

推荐用户记录方式：

1. 开始任务前记录一次快照。
2. 完成任务后记录一次快照。
3. 中间每个明显 turn 导入 transcript 或日志。
4. 每周查看报告，观察不同任务类型的消耗。

校准目标：

- 找出不同 task type 的合适 multiplier。
- 找出不同模型的额度消耗差异。
- 找出长上下文任务是否明显变贵。
- 判断请求数量是否能作为个人额度消耗的粗略代理。

## 12. MVP 开发计划

### P0: 项目骨架

- 创建 CLI 入口。
- 实现 `.codex-usage/` 初始化。
- 实现 JSONL 读写。
- 实现 ID 生成。
- 实现 config 加载。

验收：

- `codex-usage init` 可生成存储目录和配置。

### P1: 任务组与快照

- 实现 `group create`。
- 实现 `group list`。
- 实现 `snapshot`。
- 校验百分比范围。
- 支持通过 name 或 id 查找 task group。

验收：

- 可以创建任务组。
- 可以记录开始和结束用量。
- JSONL 内容可读、可手动修改。

### P2: Token 估算与 turn 导入

- 接入 `tiktoken`。
- 实现 `turn add --file`。
- 实现 `turn add --stdin`。
- 支持 `--kind`。
- 支持 `--requests` 和 `--tool-calls`。
- 根据 task type 计算 effective tokens。

验收：

- 可以导入 transcript。
- 可以生成 turn record。
- visible token 与 effective token 字段正确写入。

### P3: Report

- 实现 `report`。
- 支持 `--group`。
- 汇总 turns、requests、tool calls、visible tokens、effective tokens。
- 根据同一任务组最早和最晚 snapshot 计算 usage delta。
- 输出表格。

验收：

- 可以看到每个任务组的估算用量表。
- usage delta 为 0 或缺失时，报告清楚标记为 unavailable。

### P4: 筛选与导出

- 支持 `--since`。
- 支持 `--from` / `--to`。
- 支持 `--model`。
- 支持 `--mode`。
- 支持 CSV 导出。

验收：

- 可以按时间、模型、执行模式查看报告。
- 可以导出 CSV 做进一步分析。

## 13. 错误处理

必须处理：

- 未初始化存储目录。
- task group 不存在。
- 快照百分比不在 0 到 100。
- 同时传入冲突参数，例如 `--usage` 和 `--remaining` 的值不一致。
- 文件不存在。
- `tiktoken` 未安装。
- usage delta 为 0。
- usage delta 为负数。
- JSONL 某行损坏。

错误信息应说明下一步怎么修。

示例：

```text
Task group "repo-refactor" was not found. Run `codex-usage group list` to see available groups.
```

## 14. 报告文案要求

报告必须包含免责声明：

```text
All token values are estimated from locally visible text. They are not OpenAI billing tokens or Codex internal token counts.
```

中文版本：

```text
所有 token 数值均基于本地可见文本估算，不代表 OpenAI 账单 token 或 Codex 内部真实 token。
```

## 15. 推荐技术栈

MVP 推荐：

- Python 3.11+
- `argparse` 或 `typer`
- `tiktoken`
- JSONL 本地存储

理由：

- Python 对文本处理和 CLI 开发足够直接。
- `tiktoken` 官方可用。
- JSONL 方便调试和手动修复。

后续可选：

- SQLite：更复杂查询。
- Rich：更好看的终端表格。
- Textual：终端 UI。
- OCR：读取截图中的订阅用量。
- Browser automation：读取可见 UI 中的用量文本。

## 16. 验收场景

### 场景 1: 记录一次简单问答

```bash
codex-usage init
codex-usage group create "quick-chat"
codex-usage snapshot --group quick-chat --usage 10
codex-usage turn add --group quick-chat --file chat.md --task-type simple_chat --requests 1
codex-usage snapshot --group quick-chat --usage 10.2
codex-usage report --group quick-chat
```

期望：

- 报告显示 1 个 turn。
- usage delta 为 0.2%。
- token 字段有 estimated 标记。

### 场景 2: 记录一次代码任务

```bash
codex-usage group create "repo-refactor" --label code --label refactor
codex-usage snapshot --group repo-refactor --usage 42
codex-usage turn add --group repo-refactor --file transcript.md --model codex-max --mode local --task-type medium_code_task --requests 4 --tool-calls 12
codex-usage snapshot --group repo-refactor --usage 47
codex-usage report --group repo-refactor
```

期望：

- 报告显示 tool calls。
- visible tokens 和 effective tokens 都有值。
- visible tokens per 1% 被计算出来。

### 场景 3: usage delta 缺失

```bash
codex-usage group create "research"
codex-usage turn add --group research --file notes.md
codex-usage report --group research
```

期望：

- 报告仍显示 token 和 turn。
- usage delta 显示 unavailable。
- 不崩溃。

## 17. 后续迭代

### v0.2

- 支持自动解析 Codex transcript 格式。
- 支持 Rich 表格。
- 支持 CSV 导出。
- 支持更细粒度的文本类别统计。

### v0.3

- 支持 SQLite。
- 支持趋势图。
- 支持任务组对比。
- 支持自动建议 multiplier。

### v0.4

- 支持截图 OCR 读取用量百分比。
- 支持浏览器或桌面 UI 自动读取用量文本。
- 支持后台监控会话区间。

## 18. 实现原则

- 先保证记录准确，再追求自动化。
- 先做 CLI，不做 UI。
- 所有估算字段都要明确命名。
- 不把估算值包装成真实值。
- 数据存储要可读、可手动修复、可导出。
- 报告要服务于决策：帮助用户理解什么任务消耗更高。

## 19. 开发完成定义

v0.1 完成时，用户应能：

- 初始化本地使用记录目录。
- 创建任务组。
- 手动记录订阅用量快照。
- 导入一段 transcript 或日志。
- 使用 `tiktoken` 估算可见 token。
- 生成任务组报告。
- 看到每 1% 订阅额度对应的估算可见 token。
- 明确知道这些 token 不是 OpenAI 真实账单 token。

## 20. Codex 本地日志统计

新增 `codex` 命令直接读取本机 Codex session 日志：

```bash
codex-usage codex report --today
codex-usage codex report --date 2026-05-10
codex-usage codex report --lang zh
codex-usage codex chart --today --lang zh --output usage.html
codex-usage codex export --date 2026-05-10 --format csv --output codex-logs.csv
```

默认读取：

```text
~/.codex/sessions/**/*.jsonl
~/.codex/archived_sessions/*.jsonl
```

统计字段：

- input tokens
- cached input tokens
- non-cached input tokens
- output tokens
- reasoning output tokens
- total tokens
- primary / secondary rate-limit used percent

时间窗口统计按每个 session 的累计 `token_count` 做差值，不重复累加中间事件。

图表报告生成静态 HTML 文件，使用内联 SVG，不依赖 Node.js、浏览器服务或外部 CDN。
