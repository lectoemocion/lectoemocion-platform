# Player shell by screen

**Priority:** medium — the shell's three biggest files change together and every screen is tested through one root; no fix commits in the last 90 days, so this is cost, not breakage
**Gate:** decision:Split the player shell into one component + stylesheet + test per screen, with all state staying in `App`?

**Goal:** Give each screen of the player shell its own component file, stylesheet and test, so a change to one screen touches that screen's three files instead of the same three 1,500–2,300-line files every time.

## Context

As of `ce99351` (2026-11-21, re-checked 2026-11-22: `origin/main` has not moved).

- `apps/player-web/src/app/App.tsx` is 1,573 lines. `App()` is about 880 of them and
  holds every `useState`/`useReducer`/`useEffect` of the shell. Below it, in the same file,
  sit components that take props only and capture nothing from `App`: `TabBar`, `Tab`,
  `StarAward`, `AnimalBook`, `Chests`, `Reveal`, `WorldNode`, `stateLabel`, and four icons
  (`LockIcon`, `GamesIcon`, `ShelfIcon`, `PawIcon`), plus two small helpers that travel
  with their only user (`describeProfileFailure`, `NEEDS_ROSTER_NOTE`).
- `apps/player-web/src/styles.css` is 2,274 lines, 212 top-level class rules, imported once
  from `src/main.tsx`. It is already cut by banner comments into 14 sections (playing,
  world, stars, the sections, who is playing, the adult gate, adding and editing, the adult
  area, the book, reward, the prize settings, the goal form, a gift's form, a gift in one's
  words). Only four class blocks appear under more than one banner (`.profile-form`,
  `.prize-form`, `.prize-settings`, `.world`), so the file is nearly separable by block name.
- `apps/player-web/src/app/App.test.tsx` is 2,015 lines: 96 tests in nine `describe`
  blocks (the world shell 25, the book of names 4, the three sections 8, the book of
  animals 8, the chest 9, the letriestrellas 10, the prize meter 6, reaching the goal 9,
  profiles 17), all rendered through `<App />`, sharing about 200 lines of helpers at the
  top of the file.
- Co-change: in the 90-day window both files changed in 3 commits, the same 3
  (`a35b8db` feat(player): give the world its chrome sounds; `2dead92` feat(player): give
  the award screen the letriestrella art; `7d7b935` refactor(world): give the card back the
  fifth it reserved for its label). Over the whole history `App.tsx` has 42 commits,
  `styles.css` 44, and 28 commits touch both.
- The split has already started and stopped half-way: `Gift.tsx`, `ProfileMenu.tsx`,
  `AdultGate.tsx`, `PrizeReadout.tsx` and `adult/` are their own files (`8786d9a`
  refactor(player): give the prize readout its own file and a flare; `32e623a`
  refactor(player): move the drawn icons out of App), most with a colocated test. Their
  styles never followed — every one of them is still styled from `styles.css`.
- Two numbers that must agree live in different files: `STAMP_VISIBLE_MS` in `App.tsx` says
  it "agrees with the `book-stamp` animation in `styles.css`".
- More screens are coming: `docs/plans/ready/backend-and-adult-auth.md` adds
  `apps/player-web/src/auth/` React screens to this shell.

What must not move, and why:

- **State stays in `App`.** The comments in `App()` say so for stated reasons: `awarded`
  and `revealed` cannot be derived (the total is banked, and the reward stops being pending,
  before the screen appears); `arrival` "lives here rather than on the world screen because
  the world is unmounted for the whole ceremony"; `detour` is one fact for two screens; and
  the two arrival effects are order-dependent ("Effects run in the order they are written,
  so the reset lands first").
- **`src/app/adult/` and `src/app/AdultGate.tsx` keep their paths.** `scripts/rules.mjs`
  pins `ADULT_AREA = "apps/player-web/src/app/adult/"`, and the header of
  `scripts/check-adult-gate.mjs` (a hard stop) names `AdultGate.tsx`'s path. Nothing in this
  plan touches `scripts/`.
- **PR #2** (`feat/incentives`, opened 2026-08-07, not updated since, CONFLICTING) edits
  `App.tsx` and `styles.css`. `docs/decisions/0012-prizes-and-the-star-meter.md` says "That
  PR is closed" and lists its coupon shop under rejected alternatives, but on GitHub it is
  still open. It is not a reason to wait: nothing from it is meant to land. Closing it is a separate
  click for a person and is reported with this pass, not part of this plan.

## Design

Chosen shape — **B, the shell as a router over prop-driven screens**:

1. **One file triple per screen, flat in `src/app/`**: `X.tsx`, `X.css`, `X.test.tsx`,
   the convention `Gift.tsx`/`Gift.test.tsx` already follows. Each component imports its
   own stylesheet (`import "./X.css"`). New: `StarAward`, `Chests`, `Reveal`, `AnimalBook`,
   `TabBar`, `WorldNode`, and `World` (the final `return` of `App()`: the row of cards, the
   reward corner, the tab bar, the paw button, and the two declared layers). Existing
   components (`Gift`, `ProfileMenu`, `AdultGate`, `PrizeReadout`, `adult/*`) get the
   stylesheet they are missing. The four icons join `icons.tsx`. `STAMP_VISIBLE_MS` moves
   next to the `book-stamp` keyframes' owner, `AnimalBook`.
2. **`styles.css` keeps only what no screen owns**: `:root`, `body`, `main`,
   `.profile-failure`, `.visually-hidden`, `.button`, the playing frame, and any block used
   by more than one component. The rule is ownership by block name: a block used by one
   component goes to that component's file; otherwise it stays.
3. **Which screen is on becomes one pure function.** Today it is a chain of early returns
   whose order is the invariant ("exactly one screen is on at a time", "a ceremony does not
   interrupt a ceremony" — the first from the comment above `App()`, the second from
   `apps/player-web/AGENTS.md`). Illustrative, not final:

   ```ts
   // src/app/screen.ts
   type Screen =
     | { kind: "failure"; message: string } | { kind: "loading" }
     | { kind: "playing"; active: ActiveChapter } | { kind: "award"; amount: number }
     | { kind: "reveal"; reward: PendingRewardView } | { kind: "chests"; reward: PendingRewardView }
     | { kind: "gift"; prize: Prize } | { kind: "adult" } | { kind: "world" };
   export function screenOf(input: ScreenInput): Screen
   ```

   `App` renders with a `switch` ending in `assertNever`, the repo's own rule for unions.
   The order of the cases in `screenOf` is copied from today's returns, unchanged.
4. **`App()` keeps every hook, in today's order.** No state, effect or callback moves into
   a screen or into a custom hook in this plan.

Expected result: `App.tsx` drops from 1,573 to roughly 900 lines (the 530 lines of leaf
components and the roughly 160-line world render leave; the state and its comments stay);
`styles.css` from 2,274 to a few hundred.

Builds before `docs/plans/ready/backend-and-adult-auth.md` if that is still unstarted, so
its screens arrive in the new layout; it does not depend on it either way.

`docs/plans/ideas/take-home-animal-book.md` adds one block to the adult area and a
`@media print` block to `styles.css`. Neither plan waits for the other: whichever builds
second follows the layout it finds.

**Alternatives considered:**

- **A — move only the leaf components, leave the render chain and the world JSX in `App`.**
  Smallest diff, but the world screen (the busiest one) stays inside `App`, and the screen
  order stays testable only through a full render.
- **C — per-screen state: custom hooks (`usePlayerData`, `useGiftCeremony`) or one shell
  reducer.** Shrinks `App()` most, but moves effects whose order is documented as
  load-bearing and state that is in `App` on purpose. A behaviour risk in a refactor; can be
  its own plan later, on top of B.
- **A folder per screen (`src/app/world/World.tsx`).** Loses to flat files: `src/app/world/`
  would sit beside `src/world/` (progression) under the same name, and today a folder under
  `app/` means one thing — a gated area with one door (`adult/`).
- **CSS Modules.** Changes every class name the tests and `e2e/player.spec.ts` query; a
  behaviour-visible change with no gain over plain colocated files.

**Testing strategy:**

- `App.test.tsx` is split along its nine `describe` blocks into one file per flow; the
  shared helpers move to one harness module under `src/test/`. Tests are moved, not
  rewritten, in a first commit that changes nothing else.
- Tests that only assert what a leaf draws from its props (the builder decides per test,
  mostly in "the book of animals" and "the chest a chapter is worth") move to that
  component's own test file and render the component directly. Flows that cross screens
  (finish → stars → chests → reveal → stamp → gift) stay rendered through `<App />`.
- New `screen.test.ts`: a table over `screenOf` covering every precedence pair, including
  "gift is held while `stamping` is set". App-level tests that prove only precedence are
  replaced by it, not kept beside it.
- The CSS move has no unit test (Vitest does not load stylesheets by default). Its checks
  are `pnpm test:e2e` (`e2e/player.spec.ts`, phone and classroom layouts) and a before/after
  comparison of the built stylesheet in `apps/player-web/dist/assets/`: the same rules must
  be present. Import order now decides the order of rules in the bundle. Today `src/main.tsx` imports
  `./app/App` before `./styles.css`, which is harmless while `App` pulls in no CSS; once
  components import their own stylesheets, `./styles.css` must move above the `App` import
  or every screen's rules land ahead of the base rules. Beyond that, any two rules of
  equal specificity that land in different files and match the same element must be checked
  by hand — the four cross-banner blocks listed above are where to look first.
- `pnpm check` and `pnpm test:e2e` green before and after each step.

## Paths

As of 2026-11-22.

- `apps/player-web/src/app/App.tsx`
- `apps/player-web/src/app/App.test.tsx` (split into per-flow files, names decided at build)
- `apps/player-web/src/styles.css`
- `apps/player-web/src/main.tsx` (the order of its two imports)
- `apps/player-web/src/app/screen.ts` (new), `screen.test.ts` (new)
- `apps/player-web/src/app/World.tsx`, `WorldNode.tsx`, `TabBar.tsx`, `StarAward.tsx`,
  `Chests.tsx`, `Reveal.tsx`, `AnimalBook.tsx` (new), each with `.css` (new) and, where
  tests move, `.test.tsx` (new)
- `apps/player-web/src/app/Gift.css`, `ProfileMenu.css`, `AdultGate.css`,
  `PrizeReadout.css` (new); a stylesheet or stylesheets inside
  `apps/player-web/src/app/adult/` (new)
- `apps/player-web/src/app/Gift.tsx`, `ProfileMenu.tsx`, `AdultGate.tsx`,
  `PrizeReadout.tsx`, `adult/index.tsx`, `adult/PrizeSettings.tsx`, `adult/PrizeForm.tsx`
  (one added `import "./X.css"` each)
- `apps/player-web/src/app/icons.tsx`
- `apps/player-web/src/test/` (new harness module)
- `apps/player-web/src/app/ceremony.ts`, `bookShape.ts`,
  `scripts/generate-chrome-sounds.mjs`, `apps/player-web/public/world/PROVENANCE.md`
  (comments and prose that say `styles.css`; none is a hard-stop file)
- `apps/player-web/AGENTS.md` (the "declared layers … in `App.tsx`" pointer, the Layering
  block, the Tests section)

Not touched: `scripts/check-*.mjs`, `scripts/rules.mjs`, `scripts/rules.test.ts`,
`apps/player-web/src/app/adult/` file locations, `AdultGate.tsx`'s location.

## Approval

**Needs your yes** — the `Gate` question; recommended answer: yes, because the split
changes nothing a child or adult sees and every further screen (the auth screens are next)
otherwise lands in the same three files. The repo's `AGENTS.md` has no `## Approval`
section, so a refactor is not pre-approved here.

PR #2 does not block this: it is the coupon shop ADR 0012 rejected, and only needs closing
so GitHub matches the record.

No hard stop is touched: the plan changes no file under `scripts/`, no `*.rules` and not
`firebase.json`.

## Out of scope

- Moving any state, effect or callback out of `App()` (alternative C).
- Any change to what a screen looks like, says or does; any class rename.
- Moving `adult/` or `AdultGate.tsx`, or changing any guardrail.
- Splitting `e2e/player.spec.ts` (1,313 lines).
- Lazy-loading screens or changing how the bundle is cut.
