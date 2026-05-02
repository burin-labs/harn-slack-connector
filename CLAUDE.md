# CLAUDE.md - harn-slack-connector

Pure-Harn Slack connector package for Events API, Socket Mode, and Web API calls.

Shared Harn connector authoring rules live in the canonical guide:

- https://github.com/burin-labs/harn/blob/main/docs/src/connectors/authoring.md

Keep this file limited to provider-specific notes and local hazards. Add shared connector guidance
to the Harn guide first.

## Provider Notes

- Slack Events API expects an HTTP response within 3 seconds. Keep `normalize_inbound` CPU-only:
  verify HMAC, parse JSON, and return the ack/result without outbound network work.
- Events API signatures use `x-slack-signature` plus `x-slack-request-timestamp`; enforce the replay
  window before accepting a signed payload.
- URL verification returns an immediate response with the challenge. Socket Mode receives the same
  inner payload shape after envelope ack.
- Bot tokens are for Web API calls; app-level `xapp-` tokens are for Socket Mode connections.
