# Transcript Formats

本文件记录 `week-in-rewind` 需要识别的常见聊天记录格式：记录位置、字段、时间格式、标题与引用 id 取法。实际字段可能随工具版本变化，执行时以文件内容为准。

通用规则：会话没有现成标题时，一律用首条有效用户消息概括成不超过 10 个汉字作为标题。

## Cursor

- 常见位置：当前项目上下文会提供 `agent-transcripts` 目录；跨项目时可从该目录推断 Cursor projects 根目录，再枚举同级项目。
- 常见文件：`agent-transcripts/<uuid>.jsonl`。如果存在 `agent-transcripts/<uuid>/<uuid>.jsonl`，也可作为兼容格式读取。
- 父级记录：`subagents` 下的记录是子代理的过程记录，不是父级会话本身。
- 标题：优先使用 transcript 元数据或系统提供的聊天标题；没有时按通用规则生成。
- 时间：消息内容中的 `<timestamp>` 或结构化时间字段。
- 项目：优先使用系统上下文、transcript 元数据或消息中的 workspace/project 路径。
- 引用 id：使用父级 transcript 文件名去掉 `.jsonl` 后的值。

## Codex

- 常见位置：`~/.codex/sessions/YYYY/MM/DD/rollout-<time>-<uuid>.jsonl`，`~` 为用户主目录；如果设置了 `$CODEX_HOME`，以 `$CODEX_HOME/sessions` 为准。
- 候选定位：先按本周涉及的年月日目录筛选，再读取内容确认。
- 会话元数据：文件首行通常是 `type:"session_meta"`，包含 `cwd`（项目路径）和 `id`（会话 uuid）。
- 用户消息：格式是 `type:"response_item"` 且 `payload.role` 为 `"user"`，文本在 `payload.content[]` 中 `type` 为 `input_text` 的 `text` 字段。每条会话首条 role:user 通常是 `<environment_context>` XML（cwd/shell/日期等环境信息），属噪声，提取首条有效用户消息时必须跳过。
- 时间：时间戳通常为 UTC，带 `Z`，需换算到本地时区。
- 效率：`session_meta` 可能包含很大的 `base_instructions`，不要默认整文件全量读入；优先用 Grep 检索 `"role":"user"`、`"role":"assistant"`、`session_meta` 等关键行。
- 引用 id：使用 `session_meta.id`；如果缺失，使用文件名末尾的 uuid。

## Claude

- 常见位置：`~/.claude/projects/<project-key>/*.jsonl`。
- 候选定位：按文件所在项目和消息内时间筛选。
- 会话标识：优先使用消息或会话中的 `sessionId`、`uuid`、`id` 等字段；没有稳定字段时，使用文件名去掉 `.jsonl` 后的值。
- 用户消息：通常需要在 JSONL 中识别 role/type 为 user 的消息；字段名可能是 `message`、`content` 或嵌套的 content blocks。
- 助手消息：用于提取修改、分析结论、验证结果和未完成事项；工具调用输出只在能支撑结论时使用。
- 时间：常见为 ISO 时间戳，按本地时区理解。
- 项目：优先从 `cwd`、项目路径字段或 `<project-key>` 还原；无法确认时，用首条用户消息和文件来源辅助判断。

## Trae

- 常见位置：`~/.trae-cn/memory/projects/<project-key>/<YYYYMMDD>/session_memory_<session_id>.jsonl`；日期目录下可能有 `topics.md`（主题级目标与进展概要）。项目根目录的 `project_memory.md` 是项目规则，不是工作记录。
- 当前项目：系统上下文会直接提供当前项目的 memory 路径；跨项目时枚举 `~/.trae-cn/memory/projects/` 下的所有 `<project-key>`。
- 内容：每行一个 JSON 摘要对象，常见字段 `intent`（用户意图）、`actions`（执行动作）、`outcome`（结果）、`learned`（经验教训）；属于按消息聚合的摘要，不是逐字 transcript，足以还原工作内容，但不含完整工具输出。
- 检索：单个文件很小，可直接读取；跨项目批量筛选用 Grep 按 `"intent"`、`"message_summary_time"` 定位。
- 标题：记录没有会话标题，按通用规则用首条 `intent` 生成。
- 时间：目录名为本地日期 `YYYYMMDD`；行内 `message_summary_time` 为本地时间 `YYYY-MM-DD HH:mm:ss`，以此判断是否在本周范围；同一次会话跨天会被拆到多个日期目录，按 `session_id` 归并。
- 会话标识与引用 id：文件名中的 `<session_id>`；同源去重也按它。
- 项目：`<project-key>` 是工作区路径分隔符替换为 `-` 后再加 `--p2-<hash>` 后缀，路径本身含 `-` 时无法唯一还原，仅作线索；无法确认时用记录内容辅助判断。
- 注意：正在执行的会话记录可能尚未写入，以已存在的文件为准。

## Cross-Source Deduplication

- 同源去重：Cursor 按父级 transcript uuid；Codex 按 `session_meta.id`；Claude 按 `sessionId`/`uuid`/文件名；Trae 按 `session_memory_<session_id>` 文件名中的 session id。
- 跨源：各平台的会话 id 无法可靠对齐，是否为同一任务需按语义判断（归并规则见 SKILL.md 工作流第 5 步）。
