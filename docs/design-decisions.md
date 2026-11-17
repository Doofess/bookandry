# Design decisions

The architecture, the non-goals, and the design decisions that would look like bugs without an explanation.

## Architecture

One-way layering, enforced by lint:

```
types → lib → i18n → domain → {db, services} → features → app
                              store/, components/  ← leaves
```

- **`lib/`** — money, dates, ids, colour. Pure. No clock: `new Date()` and
  `Date.now()` are banned outside `lib/clock.ts`, so nothing can silently depend
  on when the tests happen to run.
- **`domain/`** — occurrence expansion, the calculators, the chart generator,
  form parsing. Pure, and unaware that IndexedDB exists.
- **`db/`** — Dexie, repositories, the transfer format, cross-tab locks.
- **`services/rates/`** — HTTP only, zero Dexie. Split from `db/repos/rates.ts`
  to break a cycle the original design guaranteed.
- **`features/`** — screens and their hooks. `useLiveQuery` lives here and in
  route containers only, never in a leaf: one IndexedDB subscription per feed
  row would be thousands of observers on a list being flung past.

### Three memo layers, deliberately different dependencies

5,000 transactions across ten years of monthly series is ~600,000 occurrence
objects. The answer is not to compute it faster, it is to refuse to compute it:

| Layer | Work                   | Depends on                                    |
| ----- | ---------------------- | --------------------------------------------- |
| A     | Dexie interval query   | profile + window — **not filters**            |
| B     | occurrence expansion   | candidates + window + today — **not filters** |
| C     | filter, group, flatten | everything, over ≤ 2k items                   |

Typing in the search box must not touch IndexedDB and must not re-expand a
single series. Collapse the layers and every keystroke re-queries the database.

No worker: structured-cloning 600k objects costs more than computing them, so it
would only help in the case where it also cannot help.

`src/domain/scale.test.ts` runs this at the size it is claimed about — 5,000
series that have been recurring for ten years — because every other test in the
suite runs at a few dozen rows, where the distinction cannot fail. It asserts the
invariant rather than a frame rate: a month costs one occurrence per series, not
one per month lived. The regression it exists for is an expander that walks
forward from the series start, which returns the **right answer** ~120× too
slowly and leaves every correctness test in the repository green.

### Money

Integer minor units everywhere. Exponents come from a **frozen, committed ISO
4217 table**, never from `Intl` — `resolvedOptions().maximumFractionDigits`
returns CLDR minor units, which vary by ICU build. `amountMinor` is storage: two
browsers disagreeing about the exponent means the same integer denotes two
different amounts.

Rounding is **half away from zero**, matching `Intl`'s default `halfExpand`, so
stored and displayed values never disagree, and `convert(−x) === −convert(x)`
holds. One rounding per conversion, never an intermediate.

### Dates

Hand-rolled string maths on `"YYYY-MM-DD"`, no dependency. `date-fns` is
actively wrong here: `new Date("2026-01-31")` parses as **UTC**, so west of UTC
the local getters read January 30 before any arithmetic happens. Temporal is not
viable yet.

Two clocks, and conflating them is a real bug: `todayLocal()` reads the
**device-local** calendar date, while every `Intl.DateTimeFormat` call on a
stored date passes `timeZone: "UTC"` so the rendered day does not shift per
viewer.

## Non-goals

- **Budgets.** The two headline numbers are a balance and an expectation, not an
  allowance. "Remaining" was renamed to "still due" for exactly this reason.
- **Split expenses and transfers.** The product answer is a Household profile
  plus a shared payment method, offered during onboarding. "I paid you back" is
  an expense on A and an income on B, which nets correctly in "All" — at the
  cost of inflating both stat cards.
- **End-to-end encryption.** It was built (a passphrase and a recovery
  key, 2026-09-27) and taken out again two days later: a key to type on every
  new device asked too much of people for a household ledger. Signing in, with
  a second factor on the account, is the protection. The machinery for secret-
  held keys is still in `src/services/crypto` if it is ever wanted back.
- **CSV.** JSON export only; it round-trips, and a CSV of a recurring series is
  a lie about the data model.
- **Rate backfill.** A transaction's rate is snapshotted when it is saved and
  never revisited.
- **Cross-session undo.** The undo toast and the 24-hour tombstone window are
  the whole recovery story.
- **Privacy between profiles.** Profiles are _organisation, not privacy_. Profiles
  inside one account are shared with everyone who uses that account. The onboarding says
  so in as many words.
- **A localised install name.** The PWA name is "Bookandry" in both languages, and
  the brand stays Latin inside Hebrew copy too.

## Design decisions

The ones that would look like bugs without an explanation.

**The database is still called `greenback`.** The app was Greenback before it
became Bookandry (bookandry.com). Every user-visible name changed; the storage
identifiers did not. IndexedDB finds a database by name, so renaming it would
open an empty one on every existing install - the app would restart at the
first-run wizard with the user's data orphaned under the old name. The
`gb.boot` mirror and the `greenback.*` Web Lock names stay for the same
reason. New backups say `"app": "bookandry"`; import still reads
`"greenback"`, because a backup that stops restoring when the app changes its
name is not a backup.

**Occurrences reuse the parent's rate snapshot.** A recurring transaction stores
one rate, and every occurrence it emits uses it. Deriving a per-occurrence rate
would make the headline balance a function of IndexedDB cache warmth — the same
data, on the same device, showing a different number after a cache purge.

**The balance line on the chart stops at today.** Projected bills are still
drawn as dots, because they exist and hiding them makes the month look emptier
than it is. But the balance card excludes projections by construction, and a
line running past today would report a different balance from the card directly
above it.

**Monthly recurrence clamps non-stickily.** A series on the 31st yields
`01-31, 02-28, 03-31, 04-30` and **returns to the 31st** after February.
Iterative `addMonths` gives `Jan31 → Feb28 → Mar28` and loses the 31st forever.

**An expense has a TYPE, not a repeat frequency.** The five are the ones an
Israeli already uses: עסקה רגילה, עסקת תשלומים, עסקת קרדיט, הוראת קבע, הלוואה. Four of
them collapse onto recurrence the data model already had — a הוראת קבע _is_
monthly or yearly recurrence — so the mode carries the user's word for the thing
and the storage stays one shape. Income keeps the plain repeat vocabulary: a
salary arrives every month and is nobody's standing order.

**קרדיט is priced as a לוח שפיצר; תשלומים is not priced at all.** They are the
same arrangement — a purchase split over N monthly payments — differing only in
the interest the issuer adds. The rate belongs to the **card**: it is set once
in the payment method, and the form fills it in when קרדיט is chosen (and
re-fills it when the card changes), editable per purchase for a promotional
rate. The payment is the level Spitzer payment, `PV·r(1+r)^n / ((1+r)^n − 1)`
with `r` the annual rate over twelve, computed in exact integer arithmetic with
one rounding. What is stored is what the card will charge: ₪1,200 over 6 at 12%
is six payments of ₪207.06, and the form shows the total and the interest in
shekels before saving. The rate is **snapshotted** onto the plan, like an
exchange rate, so changing the card's rate next year never re-prices a plan
already running. An empty rate, or zero, makes a קרדיט plan exactly תשלומים.

**A הלוואה is the same לוח שפיצר for money borrowed.** Stored as a plan
(`plan: 'loan'`) with its own rate, typed from the loan agreement rather than
taken from a card, up to 360 payments so a mortgage fits, and paid with any
method — the bank's debit, cheques, cash. Its repayments are filed under the
built-in Loans category unless one is picked by hand, and have their own group
in the month's breakdown. The money received is not recorded: the feed tracks
what leaves the account. CPI-linked (צמוד מדד) loans are not modelled.

**A credit card bills monthly, so its purchases leave the account later.** A
credit card can carry what its company shows about it: the card company
(Isracard, Cal, Max, Amex, Diners), network, bank or non-bank card (כרטיס
בנקאי / חוץ בנקאי), level, club, the month it expires, its billing day (מועד
חיוב, usually the 2nd, 10th or 15th), credit limit, monthly card fee,
foreign-currency fee, and its קרדיט and late-payment interest (prime + margin,
or fixed). Never its number or the code on its back (`gate:pci`). With a
billing day set, a month's purchases are charged together on that day of the
next month: the Balance counts them then - the day the money actually leaves
the bank - and its breakdown shows what is bought and not yet charged. An
instalment is a purchase of its own month. The foreign-currency fee is added to
the converted amount when a purchase in another currency is saved, and kept on
the money snapshot (`feeRate`) so a reconversion applies it again. The Cards
screen shows each card's next bill, the month building up, past bills with
their purchases, and how much of the limit is used - including every
instalment still to come, as the card company counts it. That same money is
the card's **total commitments** (סך הכל התחייבויות בכרטיס): purchases waiting
for their bill plus every remaining payment of its תשלומים, קרדיט and loan
plans, as charged, with the interest inside them shown apart - the present
value of each plan's remaining payments at its own rate is the principal still
owed, and the rest is interest. The total across everyone's cards, and each
card's share of it, heads the screen. Plans are followed to their
last payment, however far off. The monthly card fee
(דמי כרטיס) is a line on the card at each month's end, from the month it was
set - worked out, not saved, so it is changed on the card - with an optional
discount for a month the card is used for at least a set amount (100% waives
it), usually for a year (`until`). It is in the feed, the balance (on the bill)
and the chart, in a shekel base only; a shared card's fee is in everyone's
view. Not modelled: cycles that close on a day other than the end
of the month.

**A bank account is written the way the bank writes it.** A payment method of
its own type (`bank_account`): the bank by its Masav code (`src/domain/banks.ts`:
the banks in Israel, the codes of banks that have merged, and the Arab and
Palestinian banks that clear through Masav, with their Arabic names), the
branch, the account number - pasting "12-689-123456" fills all three - whose
it is (personal; joint, as a couple's; or business, with partners outside the
app by name), and its overdraft limit (מסגרת עו״ש). A joint account held by
several people is shared, and is offered only to its holders.
The branch is named as it is typed, from the Bank of Israel's list of branches
on data.gov.il: the app ships a copy (`src/data/bank-branches.json`, its own
chunk, refreshed with `pnpm data:banks`) and fetches the live list - whole,
never by the number typed - only when the copy lacks a branch. A card can say
which account pays its bill (`card.accountId`); the card editor offers the
bank accounts already set up first, and each account shows the card bills it is about
to pay. Paying straight from an account is any method but תשלומים and קרדיט,
which are a card's. Not modelled: an account's balance.

**One place for how everyone pays.** The account menu lists _Bank
accounts_ (`/bank-accounts`) and _Cards and payment methods_ - the Cards
screen, which holds the credit cards and, under them, every other way to pay:
debit cards, wallets, transfers, standing orders. `/payment-methods` sends
there. Cash and cheques are built in: every install has them, for everyone,
and they can be neither deleted, renamed nor put away (`isActiveMethod` offers
them even where an older install archived one).

**"Smart import" reads files into transactions, and code checks the reading.**
It also reads the **cards and bank accounts** a file shows - a card company's
list of cards and their rates, a statement's header (`import-cards.ts`): a
card's name, last four digits, network, issuer, קרדיט rate (kept as prime plus
its margin when the page says its rates are prime-based), limit and billing
day; an account's bank, branch, number and holders. An image's are read twice
and compared, a text's are found in the text. Each is matched to the
ones already in the app - by the card's last four digits - and listed as new, as an
update, or as already there. One the file shows only part of (a card without
its billing day, an account without its branch) opens in the card or account
editor with what it did show filled in, for the person to finish; when that is
all the files held, the editor opens straight away. A statement row's card is
the one whose last four digits are printed beside it.
The sheet also takes an optional note (300 characters) about the files - "these are all my Max card transactions for September" - sent in the text of each call of that import, context only: what the files show always wins. It is kept nowhere.
The + button opens a menu: _Add manually_, or _Smart import_, which takes
screenshots (a Bit or bank notification), photos, PDFs, Excel/CSV and Word
files - picked, pasted, dropped, or shared from the phone (the installed
app is an Android share target; `public/share-target-sw.js` keeps the files
until the person says "Read"). Nothing is added before a review list: each
row ticked or not, with its reason in words, and editable. Accuracy and
cost come from doing as little as possible with the model:

- **Code first.** Spreadsheets are read by code. Exports the app knows by
  their exact headers - Max's transaction details, Discount's bank-cards
  months - have their own readers (`import-sheet-readers.ts`): the amount is
  what the card charged, the original kept beside it only for a named foreign
  currency (Discount prints that currency only in the cell's number format,
  which is kept for it); an instalment or loan row is this payment, and a
  deal's rows make one plan; Max's FX wallet gives each purchase once, at its
  shekel cost. Discount's summary of monthly totals is refused, so no month
  counts twice. Any other sheet is read from a dictionary of the headers
  Israeli exports use (`import-sheet-map.ts`); the model only names the
  columns of one it does not know, from its first rows. A PDF with a
  text layer goes as text. Only scans and screenshots go as images, sized to
  the largest the model reads without enlarging, long captures sliced.
- **Transcribe, never compute.** The model copies amounts, dates and
  currency marks exactly as printed (short keys, structured output); the
  app's own parsers turn them into money (`parseStatementAmount`, which
  refuses an ambiguous "1.234" rather than guess) and dates (day first).
- **Checks after the first read** (`import-check.ts`): a text source's
  amounts must be printed in it; a document's rows must add up to the total
  it prints - and a difference is traced to the row it points at (a missing
  or doubled row, a moved decimal point, swapped digits); an image with no
  total is read a second time by a cheaper model and the reads compared. One
  correction call re-reads only the rows that failed; a correction is taken
  only when it is printed (text) or makes the sums meet exactly. What is
  still in doubt stays unticked.
- **What is already in the app**: rows already there (any occurrence of the
  same kind and amount within two days, a plan already running, a card's fee
  the app already charges) are grouped as "looks already added" - but never
  a row on one card for an occurrence on another, and on the same card a day
  apart only for the same merchant; a category
  comes from the app's own history for that name first.
- **One call per request, a daily cap per person** (`ai_take_quota`), every
  request and answer checked by `import-wire.ts` - shared verbatim with the
  function - with card and account numbers masked. Every file is read
  afresh - a file picked again too - so a fix to the reading reaches it;
  readings older versions kept are cleared on boot (`forgetImportReads`).

**Prime is fetched, never typed.** Israeli prime is the Bank of Israel's rate
plus 1.5. The app keeps the Bank's rate on the device (`meta`, not synced),
asks the `prime-rate` relay for it at start, when the app comes back into view
or online, and hourly, and asks again on the day a decision is due; the relay
caches for fifteen minutes. A build ships its own last-known figure for a device
that has never been online, shown with its date. A loan is priced at prime plus
its margin (signed: P−0.5 is ordinary) or at a fixed rate, and keeps the rate it
was priced at, like an exchange rate. A prime-linked loan does **not** re-price
itself when prime moves: its stored repayments stay as entered, and the margin
is kept on the plan so a future version can.

What is not modelled: issuer fees, and rates that differ from the nominal
annual rate divided by twelve. The payment shown is the textbook one; a
statement that differs by an agora or two is the issuer's rounding, not a bug
here.

**The standing-order frequency has no default.** Monthly is the likelier answer
and preselecting it would still be wrong: arnona and most insurance policies are
yearly, and a default that is right most of the time is exactly how the wrong one
gets saved without ever being read. The error waits for a save attempt rather
than firing the instant the type is chosen, which would be scolding someone for
not yet having done the thing they were just asked to do.

**Instalments (תשלומים) sum to the total exactly.** ₪1,000 over 3 payments is
₪333.34 + ₪333.33 + ₪333.33, the first absorbing the remainder, which is the
Israeli card convention. Interest-free means the parts must equal the whole.

**Changing the base currency recomputes from the original amount**, never from
the stored base amount. Round-tripping is lossy: `1.000 KWD → ¥501 → 1.001 KWD`.
It forces an export first, and refuses while another tab holds the lock.

**The chart's aggregation is a mode flag on one generator**, not a second
implementation. The two modes differ in one expression — the bucket key — so
`sum(aggregated) === sum(expanded)` holds by construction rather than by being
tested into agreement.

**The plot is an LTR island inside an RTL page.** A chart is a coordinate
system, not prose: time runs left to right in Hebrew too. The legend, captions
and stat cards are prose and stay in page direction.

**Kind is encoded by shape as well as colour** — a circle for income, a diamond
for expense. Red against green is 1.6–1.8:1 dot to dot, which fails WCAG 1.4.1
outright.

**The Add pill is first in the DOM.** It is `position: fixed`, so markup order
does not move it on screen — but it decides tab order, and rendered last it sat
after every transaction row. The keyboard walkthrough found it; nothing looked
wrong. It says "Add" rather than showing a bare plus, and its menu presets the
form to an expense or income (`?add=expense|income`; `add=1` from older links
still means an expense).

**Ink on paper; money in colour.** The house is ink on warm paper (`--paper`,
#F6F2EA, #1A1510 in dark). Colour always means money: a bank, a card company,
a card or a category, each as a full-strength sheet keyed by data — a Masav
code, an issuer, a card colour key, a hue key — never by a stored colour
(`src/domain/money-key.ts`, `src/domain/category-hue.ts`). A sheet's text is a
tint of its own hue. Sapphire is the app's own colour, for its actions only.
Every pair is checked by `src/lib/contrast.test.ts` and
`src/lib/money-colours.test.ts`, which resolve the token file's aliases and
`color-mix()` the way a browser does; the colours outside it (the status bar,
the manifest, the icons) by `src/lib/brand-colours.test.ts`.

**A desktop that uses its width.** From `lg` the shell is a text rail at the
inline start — places only, Add a button at its top — and `main` is a size
container: screens lay out by the room they have (`@board/main:`,
`@wide/main:`, `@vast/main:`), so a screen beside an open panel, or inside a
pop-out, still fits. Editors are one Base UI dialog with responsive classes: a
bottom sheet on a phone, a panel at the inline end from `lg`, so resizing
never loses what was typed. Flows open as a pop-out that lays itself out by
its own width. The rail's Add opens its editor over any screen
(`src/app/shell-editors.tsx`).

**Two faces: Plex for text and amounts, Karantina for display.** IBM Plex Sans
Hebrew (300, 400, 500, 600) covers Latin, Hebrew and ₪, and its digits are
fixed-width, so amount columns line up in both languages. Karantina 700 draws
screen names and the big numerals; its digits are proportional, so it is never
used below 28px, in a column, or for a figure that counts without a box reserved
from its final value. Both are self-hosted from Fontsource and precached; only
Karantina is preloaded.

**A pop-out in the feed is one instance at screen level, anchored to its row.**
The feed is virtualised: a menu per row would unmount mid-gesture when its row
left the rendered range. One instance points at the row's control through
base-ui's `anchor` and closes on scroll or when that control leaves the page.
The category change and "Charged today" go through the edit form's own
series-edit path, and their Undo restores every row they touched.

**The balance breakdown does not list the opening balance.** The total is still
the real balance; the starting-balance line was noise in the pop-out.

**Archiving a card hides it; it never changes money owed.** Fees stop only after
the archive month, and a card still owed on stays in the household's total on Cards.

**Duplicate keeps the source row's person.** It keeps the card only if that
person can use it, and the person only while they are still in the household
(linked accounts).

**Account linking is a page in Settings, not a tab.** It is set up once, so
it sits with the account (Settings › Account linking, `/settings/linking`,
between Account and Security) and takes no tab. It was called
Household; only the words changed - the code, the `household` i18n keys, the
tables and the Supabase functions keep the old name, as the database keeps
`greenback`. `/household` and `/profiles` still land on it.

**Cash is part of the balance, not beside it.** The Cash tab (קופת מזומן) shows how
much of the balance is in the wallet: the feed's balance stays "all my money".
Cash income raises both, paying cash lowers both, and moving money between the
bank and the wallet - a withdrawal, a deposit, the amount someone started with -
moves only the cash. Those moves are a table of their own (`cashMoves`), not a
new transaction kind or payment-method type: an older build syncing alongside
this one rejects, forever, a row carrying an enum value it does not know, while
a table it does not know it pulls and drops. Its pull cursor still moves past
those rows, so a device that later updates would never fetch them: the v4
upgrade therefore resets every pull cursor once, which costs each device one
full re-download on its first launch. For the same reason a move's
`kind` is a free string and every sum ignores a kind it does not know. Income
says where it went in its own field, `destinationId` (a bank account, as a
label, or cash), not in `paymentMethodId`, which older income forms drop on
edit. A recount writes an ordinary cash income ("Cash found") or expense ("Cash
missing") tagged `cashCount`, so an older build's balance is right too, or a
zero `count` move when it matched; during the rollout an older build names
those two categories by their keys, "cashFound" and "cashMissing". Each person
has one `start` and their cash counts from it, so old cash spending cannot
drive it negative; it can still go below zero, and says so. On the start day
itself only rows written after it count, since the amount typed already holds
the rest. If two devices set up offline and both starts sync in, the earliest
counts and the screen offers to keep it and remove the others. A withdrawal or
deposit is dated from its person's start up to today, and the start stays on or
before the first of them; outside that range a move would count for nothing.
Deleting a person and moving their rows to someone else moves their cash into
that person's pot: when both have a start, the incoming start becomes a
withdrawal of the same amount, which adds to the pot just as the start did,
and their rows dated before the pot's own start stop counting. Linked accounts
share one pot - every row keeps its person, so the pot always splits back, and
unlinking takes the other person's moves off the device with their space. Not
modelled: per-account balances, cash in a foreign currency, notes and coins by
denomination, a cash move recorded on a device that was offline while the
default currency changed. The design is in
`docs/superpowers/specs/2026-10-09-cash-deposit-design.md`.

**An expense's category follows its business type.** Every expense can carry a
business type, a Merchant Category Code (`mcc`, "5812" for restaurants), and a
table files each type under a built-in category; the person can move a type into
any category of theirs (`Category.mccs`), and its past expenses with it, with
Undo. Israeli exports carry no MCC, so each row takes the first it can get: an
MCC column, the card company's sector text, the same merchant seen before, then
Claude, choosing only codes the list has. The person's own choices always win:
a category they gave a merchant beats the table, and a business type they picked
is never replaced. The list is greggles/mcc-codes (public domain), the only one
the app may ship; the Hebrew names are ours (`src/data/mcc-SOURCES.md`,
`pnpm data:mcc`). The codes alone are a small synchronous module, so a schema can
check one, while the names are their own lazy chunk. A synced row is checked
for a code's shape only, because a later build's list may have codes this one
does not; every way a code comes in - the form, a file, history, Claude -
checks the list. All three fields are optional and older builds keep them on an
edit. A cash withdrawal (6010, 6011) is not an expense in an import: it is left
unticked with a note pointing to the Cash tab. The design is in
`docs/superpowers/specs/2026-10-09-mcc-categories-design.md`.

**Banks, card companies and networks show their real logos, on white tiles.**
White in both themes, as the logos are drawn for it; rendered as `<img>`,
never inlined, so artwork from outside cannot run anything. Sources and terms
are in `src/assets/brands/SOURCES.md`.

**Amounts render in a `dir="ltr"` island with `data-value` carrying the
integer.** CLDR's Hebrew currency pattern injects two right-to-left marks and
puts the symbol after the number; left to inherit page direction, a leading
minus jumps to the far end. Every test asserts the integer, never the formatted
string — pinning `"$1,234.50"` breaks the next time ICU moves a separator.

**The font stack order is load-bearing.** `'IBM Plex Sans Hebrew', 'Plex Fallback', …`
— a face without fixed-width digits ahead of Plex, or a `:lang(he)` font rule,
breaks amount alignment. `font-synthesis: none` means a weight that is not
loaded is never faked, which is why `font-bold` (Plex 700) is banned by
`src/lib/fonts.test.ts`.
