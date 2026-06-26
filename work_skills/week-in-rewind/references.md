# Transcript Formats

本文件记录 `week-in-rewind` 需要识别的常见聊天记录格式。实际字段可能随工具版本变化，执行时以文件内容为准，优先用消息内时间判断是否纳入本周范围。

## Cursor

- 常见位置：当前项目上下文会提供 `agent-transcripts` 目录；跨项目时可从该目录推断 Cursor projects 根目录，再枚举同级项目。
- 常见文件：`agent-transcripts/<uuid>.jsonl`。如果存在 `agent-transcripts/<uuid>/<uuid>.jsonl`，也可作为兼容格式读取。
- 父级记录：只把父级 transcript 作为来源；`subagents` 下的记录用于理解过程，不作为独立来源引用。
- 标题：优先使用 transcript 元数据或系统提供的聊天标题；没有标题时，用首条有效用户消息概括成不超过 10 个汉字。
- 时间：优先按消息内容中的 `<timestamp>` 或结构化时间字段判断；文件修改时间只用于发现候选文件。
- 项目：优先使用系统上下文、transcript 元数据或消息中的 workspace/project 路径；输出时不要暴露本地路径。
- 引用 id：使用父级 transcript 文件名去掉 `.jsonl` 后的值。

## Codex

- 常见位置：`~/.codex/sessions/YYYY/MM/DD/rollout-<time>-<uuid>.jsonl`，`~` 为用户主目录；如果设置了 `$CODEX_HOME`，以 `$CODEX_HOME/sessions` 为准。
- 候选定位：先按本周涉及的年月日目录筛选，再读取内容确认。
- 会话元数据：文件首行通常是 `type:"session_meta"`，包含 `cwd`（项目路径）和 `id`（会话 uuid）。
- 用户消息：通常在 `type:"user_message"` 的 `message` 字段。
- 标题：Codex 通常没有显式标题，用首条有效用户消息概括成不超过 10 个汉字。
- 时间：时间戳通常为 UTC，带 `Z`，需要换算到本地时区后判断是否在本周范围内。
- 效率：`session_meta` 可能包含很大的 `base_instructions`，不要默认整文件全量读入；优先检索 `cwd`、`timestamp`、`user_message` 等关键行。
- 引用 id：使用 `session_meta.id`；如果缺失，使用文件名末尾的 uuid。

## Claude

- 常见位置：`~/.claude/projects/<project-key>/*.jsonl`。
- 候选定位：优先按文件所在项目和消息时间筛选；没有可靠时间字段时，文件修改时间只能作为候选线索。
- 会话标识：优先使用消息或会话中的 `sessionId`、`uuid`、`id` 等字段；没有稳定字段时，使用文件名去掉 `.jsonl` 后的值。
- 用户消息：通常需要在 JSONL 中识别 role/type 为 user 的消息；字段名可能是 `message`、`content` 或嵌套的 content blocks。
- 助手消息：用于提取修改、分析结论、验证结果和未完成事项；工具调用输出只在能支撑结论时使用。
- 标题：Claude 记录通常没有稳定标题，用首条有效用户消息概括成不超过 10 个汉字。
- 时间：常见为 ISO 时间戳；按本地时区判断是否落在本周范围内。
- 项目：优先从 `cwd`、项目路径字段或 `<project-key>` 还原；无法确认时，用首条用户消息和文件来源辅助判断。

## Cross-Source Deduplication

- 同源去重：Cursor 按父级 transcript uuid；Codex 按 `session_meta.id`；Claude 按 `sessionId`/`uuid`/文件名。
- 跨源合并：Cursor、Codex 和 Claude 无法可靠按 uuid 对齐，需按项目、模块、文件、问题描述、PR/issue、交付目标或验证结果判断是否同一任务。
- 输出原则：聊天记录是证据来源，不是周报结构；默认不输出引用，除非用户明确要求来源、依据、证明或可追溯。
