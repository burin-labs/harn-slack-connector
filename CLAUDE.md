# CLAUDE.md — harn-slack-connector

## Quick repo conventions

- File extension: `.harn`. Use `snake_case` for filenames.
- Repo directories use `kebab-case`.
- Entry point: `src/lib.harn`.
- Tests live under `tests/`. Recorded webhook fixtures live under
  `tests/fixtures/webhooks/`.

## How to test

Install the pinned Harn CLI from crates.io:

```sh
cargo install harn-cli --version "$(cat .harn-version)" --locked
harn --version
```

Run checks from the repo root:

```sh
harn check src/lib.harn
harn lint src/lib.harn
harn fmt --check src/lib.harn
for test in tests/*.harn; do
  harn run "$test" || exit 1
done
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
