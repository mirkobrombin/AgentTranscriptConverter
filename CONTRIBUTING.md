# Contributing to AgentTranscriptConverter

## Development setup

```bash
git clone https://github.com/mirkobrombin/AgentTranscriptConverter
cd AgentTranscriptConverter
foundationc build . -o build/transcript-converter
```

Run the same checks as CI before sending a change:

```bash
foundationc imports --check .
foundationc format --check .
foundationc check .
foundationc lint .
foundationc test .
```

## Code style

- Language: **Foundation Language 1** only, following the
  [Foundation Code Standard](https://github.com/mirkobrombin/foundation-lang/blob/main/docs/foundation-code-standard.md).
- Formatting: `foundationc format --write .`, 4 spaces, lines within 100 columns.
- Naming: exported declarations use `UpperCamelCase`, package-internal ones `lowerCamelCase`.
- Comments document contracts, ownership, and the behavior of the Codex and Claude Code formats.
  They do not narrate control flow.
- Keep files focused on one role (`items.fn` converts response items, `events.fn` converts
  events, `run.fn` reads and writes files). Avoid a file per declaration.
- A change to the mapping comes with a test in `tests/` and, when it changes the output, an update
  to [docs/conversion.md](docs/conversion.md).

## License

By contributing you agree your code will be released under the [MIT License](LICENSE).

## Commit messages

Commits follow Conventional Commits:

```
<type>: <subject>
```

`<type>` is one of `feat`, `fix`, `chore`, `docs`, `build`, `ci`, `refactor`, `perf`, `style`, `test`, `revert`. Keep `<subject>` short, lowercase and in English. An optional scope is allowed: `<type>(<scope>): <subject>`.

When a commit closes an issue, use `<type>[closes #ID]: <issue title>`, for example:

```
fix[closes #2]: compaction notes lose their transcript path
```

Do not add co-author or attribution trailers.
