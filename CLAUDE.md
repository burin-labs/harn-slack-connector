# CLAUDE.md — harn-slack-connector

**Read [SESSION_PROMPT.md](./SESSION_PROMPT.md) first.** It contains the
pivot context, the connector interface contract, the 3-second response
budget, and the v0 milestones.

## Quick repo conventions

- File extension: `.harn`. Use `snake_case` for filenames.
- Repo directories use `kebab-case`.
- Entry point: `src/lib.harn`.
- Tests live under `tests/`. Recorded webhook fixtures live under
  `tests/fixtures/webhooks/`.

## How to test

Until `harn add` ships
([harn#345](https://github.com/burin-labs/harn/issues/345)):

```sh
cd /Users/ksinder/projects/harn
cargo run --quiet --bin harn -- run /Users/ksinder/projects/harn-slack-connector/tests/normalize_smoke.harn
cargo run --quiet --bin harn -- check /Users/ksinder/projects/harn-slack-connector/src/lib.harn
cargo run --quiet --bin harn -- lint  /Users/ksinder/projects/harn-slack-connector/src/lib.harn
cargo run --quiet --bin harn -- fmt --check /Users/ksinder/projects/harn-slack-connector/src/lib.harn
```

## Reference Rust impl

The existing 986-LOC Rust connector at
`/Users/ksinder/projects/harn/crates/harn-vm/src/connectors/slack/mod.rs`
is the **behavior spec**. Port semantics from there.

## Upstream conventions

For general Harn coding conventions and project layout, defer to
[`/Users/ksinder/projects/harn/CLAUDE.md`](/Users/ksinder/projects/harn/CLAUDE.md).

## Don't

- Don't perform any network I/O inside `normalize_inbound`. Slack's
  3-second budget makes that a footgun.
- Don't hand-edit `LICENSE-*` or `.gitignore`.
