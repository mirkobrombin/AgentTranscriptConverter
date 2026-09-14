# Conversion

A Codex session is stored as one rollout file of JSON lines. A Claude Code session is stored as one
transcript file of JSON lines. The converter reads the rollout once, from its first line to its
last, and writes every line it can represent as one or more transcript records in the same order.
The rollout is never opened for writing.

## Input

A rollout is selected by path or by Codex session id. A session id is looked up under `sessions`
and `archived_sessions` in the Codex directory, `$CODEX_HOME` or `~/.codex`. The first line must be
the Codex session metadata. The thread name Codex shows for the session is read from
`session_index.jsonl` in the same directory when it is present.

## Output

The transcript is written to `projects/<folder>/<session>.jsonl` under `$CLAUDE_CONFIG_DIR` or
`~/.claude`. `<folder>` is the directory the session ran in, with every character other than an
ASCII letter or digit replaced by `-`, which is how Claude Code names project folders. Claude Code
shortens folder names longer than 200 characters with a hash the converter does not reproduce, so
such sessions need `--output`.

The Claude session id derives from the Codex session id unless `--session-id` is given. Every
record id derives from the session id and the source line, so the same rollout converted to the
same destination produces the same bytes.

## Records

| Codex line | Claude Code record |
|---|---|
| `session_meta` | Transcript notice naming the Codex session |
| `turn_context` | Model and working directory of the following records |
| `message` from the user, tagged as typed | User message |
| `message` from the user, tagged as injected context | User message with `isMeta` |
| `message` from the developer | User message with `isMeta` |
| `message` from the assistant | Assistant message |
| `reasoning` with a summary | Thinking block holding the summary |
| `custom_tool_call` | `tool_use` whose input is `{"input": ...}` |
| `function_call` | `tool_use` whose input is the parsed arguments, named `namespace__name` |
| `custom_tool_call_output`, `function_call_output` | `tool_result` with text and images |
| `token_usage_record` | `usage` and `id` of the assistant records of that response |
| `item_completed` inside a tool call | Kept in the `codex.items` of the tool result |
| `item_completed` for a command outside a tool call | Transcript notice |
| `compacted` | `compact_boundary`, the history Codex kept, and a compaction note |
| `task_complete` | `turn_duration` |
| `thread_goal_updated` | `goal_status` attachment |
| `turn_aborted` | Transcript notice |

Codex writes a second copy of each message, reasoning item, and user prompt as an interface event.
Those copies are counted and not converted again. `world_state` snapshots are the structured form
of context that Codex also sends as messages, which are converted. Token counts, rate limits, and
thread settings have no Claude Code record.

Text parts that are empty or contain only whitespace are left out, because Claude rejects them and
they carry no content. A content part without a Claude block type becomes text holding its JSON.

## Compactions and resume

Codex compacts a long session by replacing the earlier turns with a summary and a selection of
earlier messages. Each compaction becomes a Claude Code compaction boundary followed by the
messages Codex kept. Claude Code resumes from the last boundary, which is the point Codex resumes
from, and loads only the records after it. The earlier conversation stays in the transcript before
the boundary.

Codex encrypts its compaction summaries. Each summary is replaced by a compaction note that tells
the model where the transcript with the complete earlier conversation is stored.

## Goal

Codex keeps one goal per thread and records every change of it in the rollout. Claude Code keeps
a goal as a session Stop hook and records it as a `goal_status` attachment. On resume it re-arms
the goal named by the last `goal_status` of the loaded conversation, unless that record is met or
failed.

| Codex goal status | Claude Code record |
|---|---|
| `active`, `usage_limited`, `budget_limited` | Not met, armed on resume |
| `complete` | Met |
| `paused`, `blocked` | Cleared, as `/goal clear` records it |

Codex stops a goal when its own usage or budget runs out, which says nothing about the goal
itself, so those goals stay armed. The goal state in Codex's `goals_1.sqlite` can be newer than
the last goal event of the rollout; it is read through the `sqlite3` program when that program is
installed. The final goal state is written after the last conversation record, past the last
compaction boundary, where Claude Code looks for it.

## Memories

Codex keeps its memory as Markdown files under `memories` in the Codex directory. Claude Code
keeps memory per project, in `projects/<folder>/memory`, as files with a frontmatter and one line
each in `MEMORY.md`.

A Codex memory file is imported when one of the session's own tool calls names it, which covers
the files it read and the ones it wrote. Files mentioned only by the instructions Codex injects,
or only printed by a tool result such as a search listing, are not imported. Each imported file becomes `codex-<name>.md` with a `reference`
frontmatter and the original text unchanged, and gains a line in `MEMORY.md`. A memory file that
already exists with other content is never replaced. `--no-memories` skips this step.

## Converted sessions

A transcript that already exists is never rewritten, because Claude Code may have continued the
session in it. Running the converter again appends a goal record when the goal differs from the
transcript's last one, chained to its last record, and imports memories that are missing. The
session must not be open in Claude Code while this runs: records Claude Code writes afterwards
continue from its own last message and would leave the goal record outside the resumed
conversation.

## Session list

`claude --resume` reads only the first and last 64 KiB of each transcript. The injected context at
the start of a Codex session usually fills the first part, so the transcript ends with the records
Claude Code reads from the tail: an `ai-title` holding the Codex thread name, when there is one,
and a `last-prompt` holding the last prompt the user typed and naming the last record as the leaf
to resume from.

## Provenance

Every converted record carries a `codex` field. `codex.line` names the source line, or the line
and position inside a compaction as `line:index`. `codex.payload` holds the source fields that are
not represented by the record itself, such as message ids, phases, and the Codex system
instructions of the session.

## What cannot be carried over

Codex reasoning is encrypted with keys held by its provider. Reasoning without a readable summary
is counted in the report and left out. Encrypted compaction summaries are replaced by compaction
notes as described above.

## Guarantees

- An existing transcript is never replaced. Output is written beside the destination under a
  temporary name and renamed after the last record is flushed.
- A rollout that Codex is still writing may end in a partly written line. That line is skipped and
  reported. A malformed line anywhere else stops the conversion.
- Memory use is bounded by the longest source line, not by the length of the session.
