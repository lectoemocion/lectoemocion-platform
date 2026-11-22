# Every commit on main gets its own CI verdict

**Priority:** medium — 16 of the repo's 78 `check` runs were cancelled, every one a push to `main`; one of them hid the commit that broke e2e until a fix was pushed 2 h 17 min later
**Gate:** decision:May `check.yml` stop cancelling runs on `main`, so each pushed main commit finishes its own check (free on this public repo, about 8 runner-minutes per push)?

**Goal:** A push to `main` never ends without a CI verdict, so a red trunk names the commit that broke it and the landing script's "post-merge CI is never cancelled" assumption becomes true here.

## Context

As of `ce99351` (2026-11-21, re-checked 2026-11-22).

- `.github/workflows/check.yml` sets `concurrency: { group: check-${{ github.ref }}, cancel-in-progress: true }` for both triggers (`push` to `main`, `pull_request`). On a PR branch that is right: only the newest head matters. On `main` it means a second push while a run is still going (about six minutes today, about three in August) cancels the first commit's run. Each of the 16 cancelled runs ends within seconds of the next push's run starting.
- `gh run list -L 100` returns all 78 runs the repo has (2026-08-05 → 2026-11-21): 56 green, 6 red, 16 cancelled, all `push` on `main`; no PR run was cancelled. 13 on 2026-08-07/08, then one each on 2026-09-21 (`3b5205c`), 2026-09-30 (`4c6a1ae`) and 2026-10-19 (`7e61a46`).
- **It already cost a diagnosis.** On 2026-08-08, `32e3d80` *feat(player): give the adult panel a shape it did not have* introduced `.prize-choice input { pointer-events: none }`, which made six e2e cases block to their timeout. Its run was cancelled, and so were the next two (`22c14f8`, `77bcb85`). The first run allowed to finish was on `67d2cc1` *docs(plans): keep the on-hardware minigame verification open* — a docs-only commit, red on e2e. The next finished run, `0324d06`, was red too; five more were cancelled; `ef6d575` *fix(player): let a pointer land on the gift-place card it clicks* was pushed at 11:42 UTC, 2 h 17 min after the breaking push at 09:25 UTC, and its run finished green at 11:47. CI never said which commit broke it. (The run logs have expired — HTTP 410 — so the cause is taken from `ef6d575`'s own message and the `-S'prize-choice'` history, not from the logs. It was a real regression, not the pointer flake in `docs/plans/ideas/phaser-pointer-e2e-flake.md`.)
- **A PR's post-merge run was cancelled too:** `2fa5384` *Merge pull request #3* (2026-08-07) was cancelled by the direct push of `fd12c4c` three minutes later.
- **The landing script assumes otherwise.** `scripts/pr-land.js` (a symlink into `.agents/_shared/scripts/pr-land.js`), in `judgeIntegrationVerdict`: "A shared move that lands while the check runs is the one gap left, and the base's own post-merge CI (never cancelled) is what covers it." `.agents/land.config.json` configures no `integrationCheck`, so per the same file the post-merge run "is then the only answer". Here that run can be cancelled.
- The gap is smaller than in August but not closed: `AGENTS.md` (§ Git) still allows a direct commit to `main` for a small change, and 4 of the 8 main pushes since 2026-10-31 were direct. `main` has no branch protection (`protectedBranches: []`), `requireApprovingReview` is `false`, and `pnpm deploy:player` (`firebase deploy --only hosting`, predeploy `pnpm build`) ships whatever `main` holds without running e2e.
- **Cost:** the repository is public, so standard runners are not billed. A green run is about 1.5 min (`check`) + 5.7 min (`e2e`); a red e2e took about 16 min. The worst day on record (2026-08-08, 9 cancelled runs) would have added at most about 2.5 runner-hours.

## Design

One change, in the `concurrency` block of `.github/workflows/check.yml`: PR runs keep today's behaviour; a push to `main` gets a group of its own, so nothing can cancel it.

Illustrative:

```yaml
concurrency:
  # A PR only needs its newest head checked. A commit on main is history:
  # each one keeps its own verdict, so a red trunk names the commit that broke it.
  group: check-${{ github.event_name == 'pull_request' && github.ref || github.sha }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

The group must change, not only the flag: with a shared group and `cancel-in-progress: false`, GitHub keeps one running and one pending run per group and a newer pending run replaces the older pending one, so a burst of three pushes would still lose the middle commit. Confirmed against GitHub's "Control the concurrency of workflows and jobs" page on 2026-11-22: under the default `queue: single`, at most one run is pending per group and a newer one takes its place; `group` and `cancel-in-progress` both accept expressions over the `github` context.

Add one line to `AGENTS.md` § Autonomy contract, under "`main` is the trunk": a run on `main` is never cancelled, so the last finished run before a red one is the last known-good commit.

**Alternatives considered:**
- Only `cancel-in-progress: ${{ github.event_name == 'pull_request' }}`, same group — loses queued middle commits in a burst (see above).
- Same group for `main` with `queue: max` (GitHub's concurrency docs, read 2026-11-22: up to 100 pending runs kept instead of one) — keeps every commit's verdict but runs them one after another, so a burst of nine waits over an hour for its last verdict, and `queue: max` cannot be combined with `cancel-in-progress: true`, which the PR side needs. A group per commit is simpler and parallel.
- Forbid direct pushes to `main` (branch protection, everything through `pnpm pr:land`) — changes how Álvaro and agents work here, and a merge followed quickly by another merge would still cancel; a separate decision.
- Leave it: the last commit of a burst is always checked, so the tree is eventually judged — but that is exactly what failed on 2026-08-08, and it contradicts what `pr-land.js` relies on.

**Testing strategy:** no unit test fits a workflow setting. Verify once after landing: push two commits to `main` within a minute (the landing merge plus the plan retirement do it naturally) and confirm with `gh run list --branch main -L 5` that both runs complete; open a PR, push twice, confirm the first PR run is still cancelled.

## Paths

As of 2026-11-22:

- `.github/workflows/check.yml`
- `AGENTS.md`

## Approval

**Needs your yes** — the `Gate` question; recommended answer: yes, because runner time is free on this public repo and the one time it mattered the trunk stayed red for over two hours with no commit named. `AGENTS.md` has no `## Approval` section, so nothing here is pre-approved. No hard-stop path is touched.

## Out of scope

- The Phaser pointer flake (`docs/plans/ideas/phaser-pointer-e2e-flake.md`).
- Gating `pnpm deploy:player` on a green `main`, or adding branch protection.
- Adding an `integrationCheck` to `.agents/land.config.json`.
- Editing the shared `.agents/_shared/` copy.
