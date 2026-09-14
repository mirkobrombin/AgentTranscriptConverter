# AgentTranscriptConverter

A converter that turns [Codex](https://github.com/openai/codex) sessions into
[Claude Code](https://docs.anthropic.com/en/docs/claude-code) transcripts, written in
[Foundation](https://github.com/mirkobrombin/foundation-lang).

The whole conversation is carried over, not a summary of it. A converted session keeps every
prompt, assistant message, tool call with its complete input and output, image, readable reasoning
summary, interruption, turn duration, and compaction. Its goal stays armed in Claude Code's goal
mode, and the Codex memories it used become Claude Code memories of the project. The Codex rollout
is opened for reading only.

| Codex | Claude Code |
|---|---|
| Prompt typed by the user | User message |
| Injected context (AGENTS.md, environment, goals, developer instructions) | User message marked as context |
| Assistant commentary and final answers | Assistant messages, with the token usage of their response |
| Tool call and its output | `tool_use` and matching `tool_result`, images included |
| Command, file change, search and MCP activity inside a call | Kept with the tool result |
| Compaction | Compaction boundary followed by the history Codex kept |
| Thread goal | Goal re-armed by `claude --resume` while Codex had not completed it |
| Memory files the session used | Project memories listed in `MEMORY.md` |

## Requirements

- A Foundation compiler (`foundationc`), which needs LLVM 21
- Claude Code, to resume the converted session
- `sqlite3`, optional, to read the goal state Codex keeps in its database

## Build & Install

```sh
foundationc build . -o build/transcript-converter
install -m 755 build/transcript-converter ~/.local/bin/
```

## Usage

Pass a Codex session id or the path of its rollout:

```sh
transcript-converter 0190a000-0000-7000-8000-000000000001
transcript-converter ~/.codex/sessions/2026/01/02/rollout-2026-01-02T10-00-00-0190a000-....jsonl
```

The transcript is written to the Claude Code project folder of the directory the session ran in,
so it appears in `claude --resume` there. The printed session id resumes it directly:

```sh
cd /path/the/session/ran/in
claude --resume <claude session id>
```

`--output` writes to another file, `--session-id` chooses the Claude session id,
`--no-memories` leaves the project memory alone, and `--codex-home` and `--claude-home` replace
`$CODEX_HOME` (`~/.codex`) and `$CLAUDE_CONFIG_DIR` (`~/.claude`).

Running the converter again on a session that was already converted never rewrites the
transcript. It brings the goal and the memories up to date, so close the session in Claude Code
first.

The mapping, the limits, and the guarantees are described in
[docs/conversion.md](docs/conversion.md).

## License

MIT, see [LICENSE](LICENSE).
