
# Port blueprints: GitHub teams, users, repos, and access roles

## ⚠️ Do this first
Port's **legacy GitHub app** integration is deprecated **15 September 2026**
(per Port's own docs). Use the **GitHub (Ocean)** integration — everything
here targets Ocean's mapping syntax.

## Files
- `blueprints/githubUser.json`
- `blueprints/githubTeam.json`
- `blueprints/githubRepository.json`
- `blueprints/githubTeamRepositoryAccess.json` — the junction blueprint
- `port-app-config.yml` — the ingestion mapping

## Why a junction blueprint
GitHub's permission model isn't "team has access to repo" — it's "team has
*this specific role* (pull/triage/push/maintain/admin) on *this* repo," and a
team can hold a different role on each repo it touches. Port relations link
two entities but don't carry their own properties, so there's no way to
attach "role" directly to a `team ↔ repository` relation.

`githubTeamRepositoryAccess` solves this by being its own entity: one row per
(team, repo) pair, holding the `role` property. This gets you both directions
for free in Port's UI — open a repo and see every team with access and their
exact role; open a team and see every repo it can reach and at what level.

## Parent/child teams
`githubTeam` has a self-relation, `parentTeam` → `githubTeam`. GitHub's
GraphQL team object exposes `parentTeam`, so each child team just points at
its parent. Port automatically shows the reverse ("child teams") on the
parent's entity page — no extra relation needed.

## Import order
1. Create the four blueprints in this order: `githubUser`, `githubTeam`,
   `githubRepository`, `githubTeamRepositoryAccess` (relations need their
   targets to exist first).
2. Install/configure the GitHub (Ocean) integration.
3. Paste `port-app-config.yml` into the integration's data source config
   and resync.

## Before you resync at scale
Two lines in the mapping are my best inference from Port's public docs, not
confirmed against a live payload — check them in Port's JQ playground first:
- `.parentTeam.slug` — GitHub's GraphQL `Team` type does expose `parentTeam`,
  but confirm the exact shape your integration returns.
- `.__repository` inside the `itemsToParse` block — this is the pattern Port
  uses elsewhere to reference the parent object's context, but I couldn't
  verify the exact variable name for `itemsToParse` on a native `repository`
  resource (vs. the webhook context where this pattern is documented). If it
  doesn't resolve, Port's JQ preview panel will show you the actual field
  names available inside each parsed item.

## Suggested dashboard widgets (once data is flowing)
- Pie chart: `githubTeam` entities broken down by... actually role lives on
  the junction now, so break down `githubTeamRepositoryAccess` by `role`
  instead — gives you "how many admin-level grants exist org-wide."
- Number chart: count of `githubTeamRepositoryAccess` filtered to
  `role = admin`, to flag privilege sprawl.
- Table view on `githubRepository` with a related-entities column showing
  `githubTeamRepositoryAccess` — an at-a-glance access list per repo.