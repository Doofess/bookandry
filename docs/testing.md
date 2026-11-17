# Testing

The commands for verifying the project, what the end-to-end suite cannot check, and the three bespoke gates.

## Setup

Requires **Node 26** and **pnpm 12**. Node 25 is EOL and Node 24 entered
maintenance in October 2026; corepack was unbundled from Node 25, so pnpm is
installed directly rather than enabled.

```bash
pnpm install
pnpm dev             # http://localhost:5173
```

| Command                  | What it does                                                                          |
| ------------------------ | ------------------------------------------------------------------------------------- |
| `pnpm verify`            | typecheck, lint, format, the three gates, tests + coverage, build                     |
| `pnpm verify:full`       | the above plus a hostile-timezone run, knip, the full Playwright suite and Lighthouse |
| `pnpm test`              | unit, db and dom projects                                                             |
| `pnpm test:e2e`          | Playwright, against real builds: local-only (4173) and production (4174)              |
| `pnpm exec lhci autorun` | Lighthouse, four routes, median of three runs                                         |

`pnpm verify` is the gate at every phase boundary. `verify:full` is slower and
runs at the checkpoints.

### What the suite cannot check

`e2e/09-pwa-offline.spec.ts` takes the PWA as far as a machine can: it starts its
own preview server, installs the service worker, **kills the server**, and then
asserts the app still renders, still resolves a deep route through
`navigateFallback`, and still has its data. DevTools' Offline checkbox only lies
to the page about the network; a dead port cannot be fooled.

Two things stay manual, and neither is a gap in coverage — they are outside what
any driver can reach:

- **The install click.** The omnibox install icon, its confirmation and the
  standalone window are browser chrome, not page content. `beforeinstallprompt`
  is the signal behind that icon, and headless Chromium never fires it — so the
  spec omits it rather than ship an assertion that can only ever skip.
- **The keyboard journey.** The specs check tab order, focus rings and the focus
  trap; whether the journey makes sense without a pointer is a judgement.

Both were performed by hand against this build on **2026-09-17**, along with a
native-speaker review of the Hebrew copy. They are recorded here rather than
implied, because a reader cannot otherwise tell a check that was done from one
that was only described — and they need redoing whenever the shell, the focus
order or the copy changes.

The Colour of money screenshot matrix (`e2e/screens.spec.ts`: 8 routes at 390,
1440 and 1920, in both languages and themes) passes its overflow and console
assertions in all 96 cases. On **2026-10-10** a sample of 23 of the 96 images was
opened and reviewed by eye, covering every route, both languages, both themes
and all three widths; the other 73 were not opened. For phase 2 (with the
transaction panel and the import pop-out), on **2026-10-10**, all 24 overlay
images (`overlay-*`: 2 overlays, 3 widths, both languages, both themes) were
opened and reviewed by eye; none showed clipped Hebrew, a misplaced ₪, or
missing grain.

The sapphire redesign's screenshot matrix (`e2e/screens.spec.ts`, now with
Cards) was reviewed by eye on **2026-10-09**.

### The three bespoke gates

Lint cannot express these, so they are scripts.

- **`gate:rtl`** — bans physical-direction utilities (`ml-`, `left-`,
  `text-left`, …) across `className`, template literals, `cn`/`cva`/`clsx` and
  plain CSS, **including `components/ui/**`**. Tailwind v4 has no `corePlugins`
  to disable them with, and covering the generated primitives is the whole
  point: they are where physical classes arrive.
- **`gate:money`** — bans `parseFloat`, `.toFixed(`, `* 100` and `/ 100` outside
  `src/lib/money.ts`. Block-comment aware, with a `money-allow` escape hatch for
  the handful of lines that are arithmetic on pixels or counts rather than on
  money.
- **`gate:pci`** — bans card numbers, expiry dates, CVV, IBAN, SWIFT and routing
  numbers from `src/**` and `e2e/**`, scanning i18n JSON as well as source.
  Payment methods store a user-chosen **label** only. There is no escape hatch,
  deliberately. Two exceptions the user asked for: a card keeps its **last four
  digits** (`card.lastDigits`) - what every card company and statement prints,
  shown in the feed beside each purchase and used to match a statement's rows
  to their card - and a bank account keeps its bank, branch and account number -
  what is printed on every cheque and given to anyone who sends money to it.
  Never the rest of a card's number, a password, a code, or anything else that
  moves money.
