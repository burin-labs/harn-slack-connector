# SESSION_PROMPT.md — harn-slack-connector v0

You are picking up the v0 build of `harn-slack-connector`, a pure-Harn
Slack Events API connector. This file is your self-contained bootstrap.

## Pivot context (60 seconds)

Harn is moving per-provider connectors **out** of its Rust monorepo and
into external pure-Harn libraries under `burin-labs/`. This repo is one
of the four flagship per-provider connectors. The existing Rust impl at
`/Users/ksinder/projects/harn/crates/harn-vm/src/connectors/slack/mod.rs`
(986 LOC) is the behavior spec — port its semantics into pure Harn.

Tracking ticket: [Pure-Harn Connectors Pivot epic
#350](https://github.com/burin-labs/harn/issues/350).

## Critical timing constraint

**Slack expects an HTTP 200 response within 3 seconds of delivery.**
If the response is slower, Slack treats the delivery as failed and
retries up to 3 times. The Harn VM is fast enough for verify-and-ack
within milliseconds, but:

- `normalize_inbound` must not perform network I/O.
- The orchestrator's response semantics need to ack immediately and
  hand the normalized event to user handlers asynchronously.
- The `immediate_response` field returned by `normalize_inbound` (per
  [harn#346](https://github.com/burin-labs/harn/issues/346)) is the
  vehicle for this — return the URL verification challenge or a 200 ack
  without waiting on user handler code.

Document this loudly anywhere that anyone might be tempted to add HTTP
calls to the inbound path.

## What this repo specifically delivers

A pure-Harn module that implements the Harn Connector interface and is
loadable as the `slack` provider:

- `pub fn provider_id() -> string` returning `"slack"`.
- `pub fn kinds() -> list` returning `["webhook"]`.
- `pub fn payload_schema() -> dict` returning the canonical normalized
  event schema.
- Lifecycle: `pub fn init(ctx)`, `pub fn activate(bindings)`,
  `pub fn shutdown()`.
- `pub fn normalize_inbound(raw) -> dict` — verifies the
  `x-slack-signature` (v0 scheme), handles the
  `url_verification` challenge, and normalizes Slack event payloads to
  the canonical shape.
- `pub fn call(method, args)` — outbound dispatch covering at minimum
  `chat.postMessage`, `chat.update`, `chat.delete`,
  `conversations.history`, `conversations.replies`, `users.info`.

## Slack signature verification (the core algorithm)

```
basestring = "v0:" + x-slack-request-timestamp + ":" + raw_body_text
expected   = "v0=" + hex(hmac_sha256(signing_secret, basestring))
ok         = constant_time_eq(expected, x-slack-signature)
```

Slack publishes the signing secret per app. **Reject** if:

- Either header is missing.
- The timestamp is older than 5 minutes from now (replay protection).
- The constant-time comparison fails.

> Slack's docs use the **hex** encoding (`v0=<hex>`), not base64.
> `hmac_sha256` returns hex by default; that's correct here. Don't
> reach for `hmac_sha256_base64` for Slack — that builtin exists for
> the connectors that *do* use base64 (e.g. some Linear paths).

## What's blocked

- **[harn#346 (Connector interface contract)](https://github.com/burin-labs/harn/issues/346)** —
  the formal interface (especially the `immediate_response` return
  field used for URL verification + ack) isn't accepted yet. Match
  the function shapes listed above; expect tweaks.
- **[harn#345 (Package management v0)](https://github.com/burin-labs/harn/issues/345)** —
  needed for distribution. **Do not cut a v0.1.0 release tag until
  #345 lands.**
- **[harn#347 (Bytes value type + raw inbound body access)](https://github.com/burin-labs/harn/issues/347)** —
  Slack sends UTF-8, so `raw.body_text` works for HMAC verification in
  v0. Revisit when #347 lands.

## What's unblocked

- `hmac_sha256(key, message) -> hex_string` — Harn builtin, just shipped.
- `hmac_sha256_base64(key, message) -> base64_string` — also shipped
  (not used here, but present in the toolbox).
- `constant_time_eq(a, b) -> bool` — **use this for `x-slack-signature`
  comparison**.
- HTTP, JSON, dict, list, regex, datetime stdlib.

## v0 milestones (build in order)

### M1 — Connector interface skeleton

- Stub all interface functions in `src/lib.harn`.
- `provider_id()` returns `"slack"`. `kinds()` returns `["webhook"]`.
  `payload_schema()` returns the canonical schema dict.
- `init`, `activate`, `shutdown` manage module state in a top-level dict.
- Acceptance: `harn check src/lib.harn` exits 0; smoke test imports the
  module and calls each interface function without error.

### M2 — Signature verification + URL verification handshake

- Port signature verification from
  `/Users/ksinder/projects/harn/crates/harn-vm/src/connectors/slack/mod.rs`.
  Use the algorithm above, exactly.
- Detect `payload.type == "url_verification"` and return:
  ```harn
  {
    immediate_response: { status: 200, body: payload.challenge, content_type: "text/plain" },
    event: none,
  }
  ```
  …without invoking user handlers.
- For all other events, return:
  ```harn
  { immediate_response: { status: 200, body: "" }, event: <normalized> }
  ```
- Acceptance: `tests/normalize_smoke.harn` covers:
  - Valid `url_verification` → returns the challenge.
  - Valid `event_callback` → returns normalized event with 200 ack.
  - Tampered body → `Err`.
  - Missing signature header → `Err`.
  - Timestamp older than 5 min → `Err`.

### M3 — Event normalization

- Normalize the verified payload per event type. v0 covers:
  `message`, `app_mention`, `reaction_added`. Each produces:
  ```harn
  {
    event_type: "message" | "app_mention" | "reaction_added",
    resource_id: "<channel>/<ts>",
    channel: "...",
    thread_ts: "...",
    user: { id, name? },
    text: "...",
    occurred_at: "...",
    raw: <original event payload>,
  }
  ```
- Ignore `bot_id`-originated `message` events by default to avoid
  obvious infinite loops; expose an opt-in to surface them.
- Acceptance: `tests/normalize_events.harn` exercises 3+ recorded
  fixtures per event type.

### M4 — Outbound dispatch

- `call(method, args)` dispatch table with at least:
  - `chat.postMessage`
  - `chat.update`
  - `chat.delete`
  - `conversations.history`
  - `conversations.replies`
  - `users.info`
- Outbound calls go to `https://slack.com/api/<method>` with
  `Authorization: Bearer <bot_token>` and a JSON body. Slack returns
  `200 OK` even on logical errors; check `response.ok` and surface
  `response.error` as an `Err` when false.
- Acceptance: `tests/call_smoke.harn` exercises `chat.postMessage`
  against a mocked endpoint and asserts both happy path and
  `ok: false` error normalization.

## Recommended workflow

1. **Use a worktree per milestone:**
   ```sh
   cd /Users/ksinder/projects/harn-slack-connector
   git worktree add ../harn-slack-connector-wt-m1 -b m1-skeleton
   ```
2. **Read the Rust impl side-by-side** for HMAC + URL-verification
   details.
3. **Test the timestamp window aggressively.** It's easy to skew
   server clocks or test fixtures.
4. **Pin webhook fixtures from a sandbox workspace**, never from a
   production workspace.

## Reference materials

- Harn quickref: `/Users/ksinder/projects/harn/docs/llm/harn-quickref.md`.
- Harn language spec: `/Users/ksinder/projects/harn/spec/HARN_SPEC.md`.
- Existing Rust impl (the spec for behavior):
  `/Users/ksinder/projects/harn/crates/harn-vm/src/connectors/slack/mod.rs`.
- HMAC builtins conformance fixture:
  `/Users/ksinder/projects/harn/conformance/tests/stdlib/hmac_sha256.harn`.
- Slack request signing docs:
  <https://api.slack.com/authentication/verifying-requests-from-slack>.
- Slack Events API:
  <https://api.slack.com/apis/events-api>.
- Slack Web API method index:
  <https://api.slack.com/methods>.

## Testing expectations

- Negative-path signature tests are mandatory: missing header, wrong
  secret, tampered body, expired timestamp.
- Use `constant_time_eq` for *all* signature comparisons.
- Mock all live HTTP for v0 tests.
- Run before committing:
  ```sh
  cargo install harn-cli --version "$(cat .harn-version)" --locked
  harn check src/lib.harn
  harn lint src/lib.harn
  harn fmt --check src/lib.harn
  for t in tests/*.harn; do
    harn run "$t" || exit 1
  done
  ```

## Definition of done for v0

- [ ] All interface functions implemented and `harn check` clean.
- [ ] Signature verification uses `constant_time_eq`, with negative
      tests proving rejection of tampered + expired payloads.
- [ ] URL verification handshake returns the challenge as a 200
      text/plain immediate response.
- [ ] `normalize_inbound` covers `message`, `app_mention`,
      `reaction_added` with bot-loop-prevention default.
- [ ] `call(method, args)` covers the methods listed in M4 with
      `ok: false` → `Err` normalization.
- [ ] No network I/O inside `normalize_inbound` (verified by code review).
- [ ] **No v0.1.0 tag cut until [harn#345](https://github.com/burin-labs/harn/issues/345)
      and [harn#346](https://github.com/burin-labs/harn/issues/346)
      both land.**
