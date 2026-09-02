# Contributing

This repository does not accept external contributions.

`harn-slack-connector` is a Harn connector package maintained inside the Burin Labs fleet. Its
CI workflow, dependency configuration, and the shared agent contract in
`AGENTS.md` are projected here automatically from `burin-labs/harn-bump-fleet`.
A patch applied directly to those files is overwritten the next time that
projection runs. The repository is public so `harn add` can fetch the package
and so its checks run on hosted runners, not to collect patches.

## Where to send a bug or a request

Open an issue on the Harn repository and label it `area/connectors`:

<https://github.com/burin-labs/harn/issues>

Useful reports name the provider, the webhook payload or API method that
misbehaved, the Harn version from `.harn-version`, and what you expected
instead. Never paste bearer tokens, signing secrets, or raw provider response
bodies into an issue.

Requests for a connector that does not exist yet belong in the same place.

## If you have write access here

Read `AGENTS.md` first. It names which parts of this repository are
fleet-managed and which are yours to edit.

- Keep provider transport and payload normalization in this package. Reusable
  orchestration belongs in Harn, product approvals and persistence belong in
  Burin, and hosted tenancy belongs in Harn Cloud.
- Do not edit inside the `<!-- BEGIN HARN SHARED AGENT CONTRACT -->` markers
  in `AGENTS.md`, and do not hand-edit `.github/workflows/ci.yml` or
  `.github/dependabot.yml`. Change those at their owning repository instead.
- Run the checks the CI workflow runs before you push.

## Pull request titles

Use `[Area] Sentence case`, where the area is one of `Connector`, `CI`, or
`Docs`.

- `[Connector] Reject webhook deliveries with a stale timestamp`
- `[CI] Repin the shared Harn package workflow`
- `[Docs] Describe the poll cursor contract`

Keep the title on one line, under about 70 characters, and say what changed
rather than which files moved. Capitalize the first word after the bracket and
leave the rest in sentence case.
