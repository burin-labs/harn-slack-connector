# AGENTS.md

Pure-Harn Slack connector for Events API, Socket Mode, and Web API calls.

Shared connector authoring rules live in the Harn guide:

- [Connector authoring guide](https://github.com/burin-labs/harn/blob/main/docs/src/connectors/authoring.md)

Put shared connector guidance in the Harn guide and keep only
provider-specific notes and local hazards here.

`CLAUDE.md` points here. Edit `AGENTS.md` only.

## Provider notes

- Slack Events API requires an HTTP response within 3 seconds. Keep
  `normalize_inbound` CPU-only: verify HMAC, parse JSON, and return the
  ack/result.
- Events API signatures use `x-slack-signature` plus
  `x-slack-request-timestamp`; enforce the replay window before accepting a
  signed payload.
- URL verification returns the challenge immediately. Socket Mode receives the
  same inner payload shape after envelope ack.
- Use bot tokens for Web API calls. Use app-level `xapp-` tokens for Socket Mode.
