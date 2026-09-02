# Repository settings proposal for burin-labs/harn-slack-connector

This file proposes changes. It does not apply them. Repository and
organization settings are the founder's to change, so treat every item below
as a recommendation waiting on a decision.

Written during the org-wide repository hygiene sweep on 2026-09-01.

## What this repository is

`harn-slack-connector` is a Harn connector package for slack. It is public so that
`harn add` can resolve the package and so its checks run on hosted runners.
Every commit to date comes from the founder or the release bot. No external
contributor has opened a pull request or an issue.

## Proposals

### Turn off issues

Issues are enabled but effectively unused. Connector bugs and connector
requests are triaged on `burin-labs/harn` under the `area/connectors` label,
which is where `CONTRIBUTING.md` now sends people. Two trackers for one
subject split the history and let a report sit unread.

Proposed: disable issues on this repository. Anyone who lands on the repo is
routed to the Harn tracker by `CONTRIBUTING.md` and by the repository
description.

Cost if wrong: an existing issue link stops resolving to an open tracker.
Closed issues stay readable, so the history is not lost.

### Leave discussions off

Discussions are already off. Keep them off. There is no audience here to
serve and no one staffed to answer.

### Restrict who can open a pull request against the default branch

A public repository accepts a pull request from any fork. This repository does
not accept external contributions, so every such pull request is a
notification that ends in a decline, and each one burns hosted runner minutes
on CI.

Proposed: require approval for all outside collaborators before workflows run
on a fork pull request. That setting lives under Actions, in the fork pull
request workflow policy, and is the narrowest control that stops the runner
spend without hiding the repository.

Cost if wrong: a legitimate outside patch waits for a manual approval before
its checks run. Given the contribution policy, that is the intended behavior.

### Keep the branch protection that already exists

No change proposed. The fleet-managed CI workflow is the required check and
should stay required.

## Not proposed

- Making the repository private. `harn add` resolves the package over a public
  URL, and hosted CI on a public repository is free. Both reasons for the
  repository being public still hold.
- Archiving. The package is still bumped on every Harn release, so it is not
  dormant.
