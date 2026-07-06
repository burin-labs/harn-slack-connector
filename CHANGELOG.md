# Changelog

All notable changes to the harn-slack-connector package are documented here.
This project adheres to [Semantic Versioning](https://semver.org/).

## 0.2.1

### Added

- **Slack file artifact helpers.** `artifact_export_request(...)` returns a
  deterministic descriptor for `files.info` plus authenticated download, or a
  direct authenticated download descriptor when Slack file URLs are already
  present. File source refs now carry the same descriptor so Harn artifact
  pipelines can import Slack-hosted PDFs/media without bespoke Slack logic.
- **Modern Slack file import descriptors.** `artifact_import_request(...)`
  builds Slack's external upload flow (`files.getUploadURLExternal` →
  upload bytes → `files.completeUploadExternal`) and remote-file registration
  flow (`files.remote.add` / `files.remote.share`). The connector does not use
  deprecated `files.upload`.
- **File event normalization.** `file_created` and `file_shared` events now
  normalize into the same source/triage model as messages and reactions,
  including canonical file deep links, privacy flags, and artifact export refs.
- **`files.info` Web API dispatch.** `call("files.info", ...)` now performs the
  Slack-compatible GET request shape and returns normalized Slack API errors.

### Changed

- Bumped package compatibility to Harn 0.9.x and the pinned local CLI to
  0.9.18.

## 0.2.0

### Added

- **Actor-chain disclosure for Slack replies.** `chat.postMessage` now consumes
  Harn disclosure metadata, adds a textual byline when `chat:write.customize`
  is absent, uses Slack username/avatar customization only when that scope is
  known granted, and emits the default machine-readable AI marker through Slack
  message metadata.
- **Eval verdict Slack digest helper.** `post_eval_verdict(...)` formats an
  upstream eval-suite verdict and sends it through the existing approved
  `chat.postMessage` Web API path.
- **Slash-command and interactivity normalization.** `normalize_inbound`
  now accepts signed `application/x-www-form-urlencoded` POSTs and normalizes
  them into `slash_command` and `interactivity` (block actions / view
  submission) envelopes. These bodies are verified with the same signing-secret
  HMAC + 5-minute replay window as the Events API path (no forked verifier).
- **Agents & Assistants support.** New `assistant_thread_context_changed`
  inbound event normalization (and `assistant_thread_started` continues to carry
  its `context`), plus outbound methods `assistant.threads.setStatus`,
  `assistant.threads.setSuggestedPrompts`, and `assistant.threads.setTitle`
  (all require the `assistant:write` scope).
- **Canonical `agent_summon` dispatch contract.** `app_mention`, DM
  (`message.im`), group-DM (`message.mpim`), and `slash_command` all normalize
  into one `payload.agent_summon` envelope
  (`{ prompt, channel_id, channel_kind, thread_ts, user, team, visibility,
  trigger_kind }`) shared by harn-cloud and burin-code dispatch.
- **`chat.postEphemeral`** outbound reply helper (alongside threaded
  `chat.postMessage`).

### Changed

- **Socket Mode is now OFF by default.** `socket_mode_connect` opens a
  WebSocket only when Socket Mode is explicitly enabled
  (`enable_socket_mode` / binding / ctx `socket_mode.enabled`) **and** an
  app-level `xapp-` token is present; otherwise it returns an `Err` and the
  connector stays on the HTTP Events path. The hot inbound path
  (`normalize_inbound`) never opens a socket.
