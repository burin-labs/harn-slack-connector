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

<!-- BEGIN HARN SHARED AGENT CONTRACT: managed by harn-bump-fleet -->

## Ecosystem working agreement

- Pursue the ambitious product outcome; make the seams boring with small typed
  interfaces, explicit invariants, and deterministic projections.
- Give each behavior one semantic owner. Generate or parity-test other surfaces
  instead of maintaining competing implementations.
- Work autonomously inside approved scope. Pause for destructive, production,
  high-spend, ambiguous, or authority-expanding actions—not routine reversible work.
- Treat stop, wait, stand down, and pivot as control events for long-lived work.
- Match evidence to the claim: exercise the canonical user path, state the
  falsifier, verify liveness and recovery, and record residual blind spots.
- "Ship" means landed on main with required deploy and post-merge checks complete.

<!-- END HARN SHARED AGENT CONTRACT -->
