# harn-slack-connector

Pure-Harn Slack connector for the Harn orchestrator. It verifies inbound Events
API signatures, handles URL verification, normalizes Slack events to the Harn
`TriggerEvent` shape, supports Slack Socket Mode receive/ack loops, and
dispatches outbound Web API calls.

This package implements Harn Connector Contract v1. Use Harn CLI `0.7.50` or
newer; this repository pins the tested CLI in `.harn-version`. The canonical
connector contract reference is the Harn trigger quick reference:
https://harnlang.com/docs/llm/harn-triggers-quickref.html

Slack expects an HTTP response within **3 seconds** for Events API delivery.
`normalize_inbound(...)` is intentionally CPU-only: it verifies HMAC, parses
JSON, returns an ack/result, and performs no outbound network work.

## Install

```sh
harn add github.com/burin-labs/harn-slack-connector@v0.1.0
```

For local multi-repo development:

```toml
[dependencies]
harn-slack-connector = { path = "../harn-slack-connector" }
```

## Events API Usage

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

## Socket Mode Usage

Socket Mode uses a Slack app-level token (`xapp-...`) to open the WebSocket and
the same normalization path as Events API for the inner payload.

```harn
import slack_connector from "harn-slack-connector/default"

pipeline socket_mode_worker() {
  let conn = slack_connector.socket_mode_connect({
    app_token: env("SLACK_APP_TOKEN"),
    max_messages: 1000,
  })
  loop {
    let next = slack_connector.socket_mode_receive(conn, {timeout_ms: 30000})
    if next.type == "event" && next.event.kind == "app_mention" {
      slack_connector.call("chat.postMessage", {
        channel: next.event.payload.channel,
        thread_ts: next.event.payload.thread_ts,
        text: "Hello from Socket Mode!",
      })
    }
  }
}
```

`socket_mode_receive(...)` acks each Slack envelope before normalizing it. Tests
can pass `socket_url` and use Harn `websocket_mock(...)`; production can omit
`socket_url` and let the connector call `apps.connections.open`.

## Slack App Setup

Create a Slack app with Events API enabled for HTTPS webhooks, or Socket Mode
enabled for WebSocket delivery. Store secrets in the Harn secret provider or
pass them through environment variables during local development:

- `SLACK_SIGNING_SECRET`: Events API signing secret.
- `SLACK_BOT_TOKEN`: bot token for Web API methods.
- `SLACK_APP_TOKEN`: app-level token with `connections:write` for Socket Mode.

Required bot scopes depend on outbound calls and subscribed events:

- `app_mentions:read` for `app_mention`.
- `channels:history`, `groups:history`, `im:history`, or `mpim:history` for
  message events in those surfaces.
- `reactions:read` for `reaction_added`.
- `chat:write` for `chat.postMessage`, `chat.update`, and `chat.delete`.
- `channels:history`/`groups:history`/`im:history`/`mpim:history` as applicable
  for `conversations.history` and `conversations.replies`.
- `users:read` for `users.info`.

Minimal Slack app manifest:

```yaml
display_information:
  name: Harn Slack Connector
features:
  bot_user:
    display_name: Harn
oauth_config:
  scopes:
    bot:
      - app_mentions:read
      - channels:history
      - chat:write
      - reactions:read
      - users:read
settings:
  event_subscriptions:
    request_url: https://example.com/slack/events
    bot_events:
      - app_mention
      - message.channels
      - reaction_added
  socket_mode_enabled: false
  token_rotation_enabled: false
```

For Socket Mode, set `socket_mode_enabled: true` and create an app-level token
with `connections:write`.

## Normalized Events

Supported Events API and Socket Mode event types:

- `message`
- `app_mention`
- `reaction_added`
- `app_home_opened`
- `assistant_thread_started`

Slack retry headers are normalized onto `event.payload.retry`:

```harn
{
  is_retry: true,
  num: 1,
  num_raw: "1",
  reason: "http_timeout",
}
```

The payload also includes `metrics.slack_retry_delivery` and
`metrics.slack_first_delivery` counters as normalized values for downstream
ingress metrics. Dedupe keys stay based on Slack `event_id`, so retries do not
dispatch duplicate work when the Harn inbox is configured with `event.dedupe_key`.

## Operations

Slack automatically disables event subscriptions when delivery success remains
too low over time. Keep the verify-and-ack path fast, alert on signature rejects
and retry spikes, and keep handler/network work outside `normalize_inbound`.
When Harn Cloud managed ingress is used, configure the connector package through
`HARN_CLOUD_CONNECTORS_CONFIG` and store the webhook secret as
`slack.webhook.secret`.

Release checklist for `v0.1.0`:

- `harn check src`
- `harn lint src`
- `harn fmt --check src tests`
- `for test in tests/*.harn; do harn run "$test"; done`
- `harn connector check .`
- package install/import smoke from a clean temp consumer project
- Harn Cloud local managed-ingress load smoke using the package checkout

## Development

Install the pinned Harn CLI:

```sh
cargo install harn-cli --version "$(cat .harn-version)" --locked
harn --version
```

Run the local CI equivalent:

```sh
harn check src
harn lint src
harn fmt --check src tests
for test in tests/*.harn; do
  harn run "$test" || exit 1
done
harn connector check .
```

## License

Dual-licensed under MIT and Apache-2.0.

- [LICENSE-MIT](./LICENSE-MIT)
- [LICENSE-APACHE](./LICENSE-APACHE)
