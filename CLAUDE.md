# CLAUDE.md - harn-slack-connector

Pure-Harn Slack connector for Events API, Socket Mode, and Web API calls.

Shared connector rules are in the Harn authoring guide:

- https://github.com/burin-labs/harn/blob/main/docs/src/connectors/authoring.md

Keep this file to Slack-specific notes. Put shared connector guidance in the
Harn guide first.

## Provider Notes

- Slack Events API requires an HTTP response within 3 seconds. Keep
  `normalize_inbound` CPU-only: verify HMAC, parse JSON, and return the
  ack/result.
- Events API signatures use `x-slack-signature` plus
  `x-slack-request-timestamp`; enforce the replay window before accepting a
  signed payload.
- URL verification returns the challenge immediately. Socket Mode receives the
  same inner payload shape after envelope ack.
- Use bot tokens for Web API calls. Use app-level `xapp-` tokens for Socket Mode.
