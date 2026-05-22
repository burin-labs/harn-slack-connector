# harn-slack-connector

Pure-Harn Slack connector for the Harn orchestrator. It verifies inbound Events
API signatures, handles URL verification, normalizes Slack events to the Harn
`TriggerEvent` shape, receives and acks Socket Mode envelopes, and dispatches
Web API calls. Events also include `harn.triage.source.v1` data for Harn and
Burin Home inbox flows.

This package implements Harn Connector Contract v1. Use the Harn CLI version
pinned in `.harn-version`. The connector contract and package authoring rules
live in the
[Harn connector authoring guide](https://github.com/burin-labs/harn/blob/main/docs/src/connectors/authoring.md).

Slack expects an HTTP response within 3 seconds for Events API delivery.
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

## Events API usage

```harn
import { call } from "harn-slack-connector/default"

trigger respond on slack {
  source = {
    kind: "webhook",
    signing_secret: env("SLACK_SIGNING_SECRET"),
    bot_token: env("SLACK_BOT_TOKEN"),
    events: ["app_mention"],
  }
  on event {
    if event.event_type == "app_mention" {
      call("chat.postMessage", {
        bot_token: env("SLACK_BOT_TOKEN"),
        channel: event.channel,
        thread_ts: event.thread_ts,
        text: "Hello from Harn!",
      })
    }
  }
}
```

## Socket Mode usage

Socket Mode uses a Slack app-level token (`xapp-...`) to open the WebSocket and
the same normalization path as Events API for the inner payload.

```harn
import { call, socket_mode_connect, socket_mode_receive } from "harn-slack-connector/default"

pipeline socket_mode_worker() {
  let conn = socket_mode_connect({
    app_token: env("SLACK_APP_TOKEN"),
    max_messages: 1000,
  })
  while true {
    let next = socket_mode_receive(conn, {timeout_ms: 30000})
    if next.type == "event" && next.event.kind == "app_mention" {
      call("chat.postMessage", {
        bot_token: env("SLACK_BOT_TOKEN"),
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

## Slack app setup

Create a Slack app with Events API enabled for HTTPS webhooks, or Socket Mode
enabled for WebSocket delivery. Store secrets in the Harn secret provider. For
tests and local scripts, pass secret IDs or direct values from environment
variables:

- `slack/signing-secret`: Events API signing secret.
- `slack/bot-token`: bot token for Web API methods.
- `slack/app-token`: app-level token with `connections:write` for Socket Mode.
- `SLACK_SIGNING_SECRET`: Events API signing secret.
- `SLACK_BOT_TOKEN`: bot token for Web API methods.
- `SLACK_APP_TOKEN`: app-level token with `connections:write` for Socket Mode.

`harn connect slack` stores connector credentials under the Slack secret IDs
listed above. Runtime configuration can also pass `signing_secret_id`,
`webhook_secret_id`, `bot_token_secret`, or `app_token_secret`; direct values
remain supported for local tests and one-off scripts.

Required bot scopes depend on outbound calls and subscribed events:

- `app_mentions:read` for `app_mention`.
- `channels:history`, `groups:history`, `im:history`, or `mpim:history` for
  message events in those surfaces.
- `reactions:read` for `reaction_added`.
- `chat:write` for `chat.postMessage`, `chat.update`, and `chat.delete`.
- `reactions:write` for `reactions.add` and `reactions.remove`.
- `views.open` and `views.update` use the app's interactivity `trigger_id` or
  view identifiers rather than an additional method-specific OAuth scope.
- `chat.getPermalink` uses the event `channel` and `ts` to hydrate exact message
  permalinks outside the webhook hot path.
- `channels:history`/`groups:history`/`im:history`/`mpim:history` as applicable
  for `conversations.history` and `conversations.replies`.
- `users:read` for `users.info`; `users:read.email` for `users.lookupByEmail`.

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

## Normalized events

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

Each normalized event also carries:

- `payload.source`: canonical Slack source ref with team/channel/user IDs,
  source URL/deep link, timestamp, permalink lookup args, and source-level
  dedupe key.
- `payload.source_refs`: related message, thread, channel/DM, member, file, or
  canvas refs when present in Slack payloads.
- `payload.triage`: `harn.triage.source.v1` adapter data for Start My Day style
  inbox events, including source refs, actor IDs, mention/direct-message/thread
  hints, privacy flags, raw-payload provenance, and local/write action intents.
- `payload.provider_raw`: Slack envelope, raw event, and redacted canonical
  request headers for audit; Slack signatures are never exposed.

Webhook and Socket Mode normalization remain CPU-only. Exact Slack message
permalinks should be hydrated later with `call("chat.getPermalink", ...)` using
the supplied `payload.source.permalink_request`; write-capable outbound methods
are marked `requires_approval` in `outbound_methods()` and in triage action
intents so upstream hosts can gate them.

## Operations

Slack may temporarily disable Events API subscriptions after more than 95%
delivery failures within 60 minutes. Keep the verify-and-ack path fast, alert
on signature rejects and retry spikes, and keep handler/network work outside
`normalize_inbound`. When Harn Cloud managed ingress is used, configure the
connector package through `HARN_CLOUD_CONNECTORS_CONFIG` and store the webhook
secret as `slack/signing-secret`, or set a binding-level `signing_secret_id` or
`webhook_secret_id`.

Local verification:

- `harn connector test "$(pwd)" --provider slack`
- Harn Cloud local managed-ingress smoke using this package checkout

## Development

Install the pinned Harn CLI:

```sh
cargo install harn-cli --version "$(cat .harn-version)" --locked
harn --version
```

Run the local CI equivalent:

```sh
harn connector test "$(pwd)" --provider slack
```

## License

Dual-licensed under MIT and Apache-2.0.

- [LICENSE-MIT](./LICENSE-MIT)
- [LICENSE-APACHE](./LICENSE-APACHE)
