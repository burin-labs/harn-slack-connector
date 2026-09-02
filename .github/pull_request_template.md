## What changed

<!--
Three to five sentences in plain language. Say what behavior a caller,
operator, or downstream repo sees differently after this merges, not which
files moved. The Files tab already lists the files.

Example: "Inbound webhooks with a stale timestamp are now rejected before the
body is parsed. Previously a replayed delivery older than five minutes still
reached normalization and produced a duplicate event. Callers see a rejected
NormalizeResult with reason `stale_timestamp`. No change to the accepted
path or to the event shape."
-->

## How you verified it

<!--
What you actually ran and what happened, not a count of green tests. Name the
load-bearing behavior you are confident about and why, then the honest blind
spots and how they get covered later.
-->

## Pull request title

<!--
Use `[Area] Sentence case`. Examples:
  [Connector] Reject webhook deliveries with a stale timestamp
  [CI] Repin the shared Harn package workflow
-->
