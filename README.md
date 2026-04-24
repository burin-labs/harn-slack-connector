# harn-slack-connector

Pure-Harn Slack Events API connector for the Harn orchestrator. Verifies
inbound signatures, handles the URL verification handshake, normalizes
Slack event payloads to the canonical `TriggerEvent` shape, and dispatches
outbound Web API calls.

> **Status: pre-alpha** — actively developed in tandem with
> [burin-labs/harn](https://github.com/burin-labs/harn). See the
> [Pure-Harn Connectors Pivot epic #350](https://github.com/burin-labs/harn/issues/350).

This is an **inbound + outbound** connector implementing the Harn Connector
interface defined in
[harn#346](https://github.com/burin-labs/harn/issues/346).

> Slack expects an HTTP response within **3 seconds** of delivery. The
> Harn VM is plenty fast for the verify-and-ack path, but anything that
> reaches a network in `normalize_inbound` will violate this budget.

## Install

```sh
harn add github.com/burin-labs/harn-slack-connector@main
```

For local multi-repo development, a path dependency is still useful:

```toml
[dependencies]
harn-slack-connector = { path = "../harn-slack-connector" }
```

## Usage

```harn
import slack_connector from "harn-slack-connector/default"

trigger respond on slack {
  source = {
    kind: "webhook",
    signing_secret: env("SLACK_SIGNING_SECRET"),
    bot_token: env("SLACK_BOT_TOKEN"),
    events: ["app_mention"],
  }
  on event {
    if event.event_type == "app_mention" {
      slack_connector.call("chat.postMessage", {
        channel: event.channel,
        thread_ts: event.thread_ts,
        text: "Hello from Harn!",
      })
    }
  }
}
```

## Development

This repo is being built out by Claude Code sessions following a structured
prompt. **Read [SESSION_PROMPT.md](./SESSION_PROMPT.md) before making changes.**

Install the pinned Harn CLI from crates.io:

```sh
cargo install harn-cli --version "$(cat .harn-version)" --locked
harn --version
```

Run the local CI equivalent from this repo:

```sh
harn check src/lib.harn
harn lint src/lib.harn
harn fmt --check src/lib.harn
for test in tests/*.harn; do
  harn run "$test" || exit 1
done
```

## License

Dual-licensed under MIT and Apache-2.0.

- [LICENSE-MIT](./LICENSE-MIT)
- [LICENSE-APACHE](./LICENSE-APACHE)
