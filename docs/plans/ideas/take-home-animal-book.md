# Take-home animal book

**Priority:** medium — the default-content pilot (roadmap 9a) and the family market (roadmap 10) have nothing joining them, and this joins them with no backend
**Gate:** decision:Should the adult area get an "Imprimir el libro de animales" button that prints the playing child's animal book on one page to send home, with no name printed (a blank line to write it by hand) and a one-line LectoEmoción footer?

**Goal:** An adult can print the animal book of the child who is playing, from the adult area, so what a child wins on the classroom panel reaches the family on paper, without an account and without any data leaving the device.

## Context

As of `ce99351` (2026-11-21, re-checked 2026-11-22: `origin/main` has not moved).

- The product is "sold to schools and directly to families" (`AGENTS.md`, Product), institutions first. `docs/product/implementation-roadmap.md` stage 9a is a classroom pilot on default content ("No uploads, no child media, no personal data beyond the adult account"), which "can begin as soon as stage 3 and stage 7 are complete", so the pilot itself starts after sign-in exists even though this sheet needs none; stage 10 is the family market, "sequenced after the institutional pilot". Nothing in between carries the product from a classroom to a home.
- The collection already exists and is display-only: `AnimalBook` in `apps/player-web/src/app/App.tsx` draws `WorldView.collection` (`CollectionSlotView` in `apps/player-web/src/world/worldView.ts`: `title`, `animal`, `earned`), one place per chapter, a shadow until won ([ADR 0015](../../decisions/0015-the-animal-book.md)). It lives only on the screen of the device that played, in that browser's local storage.
- The two ideas an adult-facing report could fall under, `parents-area.md` and `progress-reports.md`, are both low priority and neither mentions paper, printing or anything a child takes home. `progress-reports.md` needs timestamps `Progress` does not record and a privacy answer; this needs neither, because it prints only what the book already shows.
- The adult area (`apps/player-web/src/app/adult/index.tsx`, `AdultArea`) is behind the one `AdultGate`, and `scripts/check-adult-gate.mjs` keeps every module in `adult/` reachable only through it. Today it holds the sound switch and `PrizeSettings`.
- `apps/player-web/src/styles.css` has no `@media print` rule, and nothing in `apps/player-web/src` calls `window.print`.
- Correction to the idea as first stated: there are no "letters won". `apps/player-web/src/app/letriestrellas.ts` says the vowel on a star is decoration, "nothing reads the letter, and no stored state has ever heard of it". The sheet therefore carries animals only.
- Art: every animal picture is a vocabulary image (`picture()` in `packages/template-catalog/src/world/index.ts` → `/vocabulary/<id>.webp`). `apps/player-web/public/vocabulary/PROVENANCE.md` records the owner's attestation that these are free-licence stock images (CC0 or equivalent) permitting commercial use without attribution, and its known limitation: no per-file licence was recorded. Printing does not need a wider licence than shipping them does, but paper handed to families is a more visible use of the same unaudited set.
- [PR #2](https://github.com/lectoemocion/lectoemocion-platform/pull/2) is still open on GitHub (last updated 2026-08-07) and touches `App.tsx` and `styles.css`. [ADR 0012](../../decisions/0012-prizes-and-the-star-meter.md) records its concept as replaced and calls it closed. It is not treated as a blocker here; closing it is a separate housekeeping step.

## Design

**What the adult sees.** One new block in the adult area, under the prize settings: a small preview of the sheet and a button, *Imprimir el libro de animales*. The button calls the browser's print dialog. Nothing is uploaded, generated on a server, or saved; the browser prints the page it already has.

**What is on the paper.** One A4 page, portrait:

- a title, *Mis animales*, and a line *Este libro es de ____________* left blank to be written by hand;
- every place of the book in world order: the animal in colour where it is won, a pale grey silhouette where it is not (pale, not the screen's solid black, so a class set does not empty a toner cartridge), with the animal's name printed under the won ones, since the paper is read by an adult with the child;
- a count, *7 de 11 animales*;
- one footer line naming LectoEmoción.

**No name is printed.** A first name is on the privacy baseline's list of data in scope (`docs/privacy/spain-eu-baseline.md`), and the 9a pilot runs with "no personal data beyond the adult account". A sheet with no name is product-authored default content plus a count, so the pilot's premise holds on paper too, and one shared classroom profile prints a sheet that is correct for every child in the class. The blank line lets a teacher or child write the name by hand, which is the school's own act, not the product's.

**Which child.** The profile that is playing when the adult area is opened. No picker.

**Responsibilities.**

- `adult/TakeHomeSheet.tsx` (new): a pure component over `readonly CollectionSlotView[]`. It takes no profile, so it cannot print a name by accident. The print call is injected as a prop (illustrative: `onPrint: () => void`), so the component test does not need a real print dialog.
- `AdultArea` gains one prop, the collection, and renders the sheet. `App.tsx` passes `view.collection` and `() => window.print()`.
- `styles.css` gains the repo's first `@media print` block: hide everything except `.take-home`, set the page size and margins, and force colour for the pictures (`print-color-adjust: exact`). Pictures are `<img>` elements, not CSS backgrounds, because browsers drop backgrounds when printing by default.

**Where it works.** In a browser with a printer behind it: the connected-computer route to a classroom panel (roadmap stage 2), a family's laptop, and a tablet's browser through the system print service. It prints only what was played on that device, since progress is local until stage 7. Inside the native shell (`apps/mobile`, a `react-native-webview`) `window.print()` is not expected to open anything; the builder verifies this on the Pixel 8 and, if so, the block is not offered there. Printing from the shell needs a native bridge and is left to the native-shell plan.

**Order with `player-shell-by-screen.md`.** Neither depends on the other. Both edit `App.tsx` and `styles.css`: if the shell split has landed, the print rules go in a `TakeHomeSheet.css` beside the component instead of in `styles.css`; if this lands first, the split carries that block out with the rest of the adult area's styles.

**Alternatives considered:**

- *Save as an image or PDF built in the page.* Works without a printer, but adds a rendering dependency and produces a file that then travels by WhatsApp or email, which is the export the privacy baseline's "no exportable media, no public sharing" leans against. The browser's own dialog already offers "save as PDF" to an adult who wants it.
- *Print the child's first name from the profile.* More personal, and it turns the sheet into local personal data on paper during a pilot whose point is that there is none. Can be revisited for the family market, where the parent is the controller.
- *A QR code that opens the child's book at home.* The strongest bridge to families, and it needs accounts and pairing (stages 4 and 7). This is what can exist before them.
- *Wait for `progress-reports.md`.* That is a different thing (how a child is doing, over time) and waits on a schema change and a privacy review.

**Testing strategy:** a component test for `TakeHomeSheet` at its props: won places show the animal and its name, unwon places show the silhouette and no name, the count is right, the button calls the injected print, and no profile name appears anywhere in the output. An addition to `adult.test.tsx` that the block is not mounted before the gate is passed. One Playwright test in `apps/player-web/e2e/player.spec.ts` with `page.emulateMedia({ media: "print" })` asserting the sheet is visible and the rest of the adult area is not, since a print stylesheet is exactly what jsdom cannot check. No guardrail changes: the component sits inside `adult/` and is imported only by `adult/index.tsx`.

## Paths

As of 2026-11-22:

- `apps/player-web/src/app/adult/TakeHomeSheet.tsx` (new)
- `apps/player-web/src/app/adult/TakeHomeSheet.test.tsx` (new)
- `apps/player-web/src/app/adult/index.tsx`
- `apps/player-web/src/app/adult/adult.test.tsx`
- `apps/player-web/src/app/App.tsx`
- `apps/player-web/src/styles.css`
- `apps/player-web/e2e/player.spec.ts`
- `apps/player-web/AGENTS.md` (only if it lists what the adult area holds)

No hard-stop path is touched.

## Approval

**Needs your yes** — the `Gate` question; recommended answer: yes, as described (no name printed, one footer line), because it is the only link between classroom play and families that needs no backend (the 9a pilot itself waits for stages 3 and 7, this does not), it adds no stored data and no dependency, and the no-name version keeps the pilot's "no child data" promise intact.

Two smaller picks ride on that yes and can be changed in the same answer: the footer says only "LectoEmoción" and the web address (no sales text on a sheet a school hands out), and unwon animals are shown as pale silhouettes, not left off.

## Out of scope

- Printing from the native shell; any bridge in `apps/mobile`.
- Any file export, sharing, email or QR code.
- The child's name, age, avatar, letriestrella total, gifts, or anything about time or difficulty: that is `progress-reports.md`.
- A printable sheet per child for a whole class in one go; profiles are capped at `MAX_PROFILES` = 8 and the pilot runs on shared ones.
- A per-file licence audit of the vocabulary art, and closing PR #2.
