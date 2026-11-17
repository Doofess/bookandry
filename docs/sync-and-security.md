# Sync and security

How accounts, keys and sync work, the threat model, and a map of the code that does it.

Account first, local underneath: you sign in (or create an account) before
anything else, and your data is your account - on every device you sign in on,
safe when one is lost. Each device keeps a full copy in the browser, so the app
is fast and works offline once signed in. See [Accounts](#accounts).

## Accounts

Required. The app works the way a bank's app does: **your data is your
account**. The first screen is the sign-in; nothing of the app is shown before
it. Signing in to an account that has data opens the app with it; creating an
account (or signing in to an empty one) leads to the setup wizard. There is no
"sync" to turn on, watch or press. Each device still keeps a full local copy —
the app stays fast and works offline — and keeps it in step with the account in
the background.

- **A new device** fetches the account's data before deciding anything, so a
  returning person is never offered the setup wizard while their data is on
  its way.
- **Data from before accounts** (a device used signed out) is added to the
  account it signs in to.
- **A build without Supabase configured** (a fork; the e2e suite's local build,
  `pnpm build:local`) has no accounts at all and is the local-only app. The
  e2e suite drives the app screens through it, since a test cannot sign in to
  a real account, and the account screens through the production build.

- **Signing out takes the data off the device.** The account keeps it. If some
  changes have not reached the account yet (offline), signing out says so
  first.
- **A device whose session ends any other way** — locked after inactivity, or
  signed out from another device — keeps the data but shows only the front
  door until the same account signs in again. Signing in with a different
  account removes it first.
- **One front door for everyone** (`src/features/account/welcome-screen.tsx`,
  `auth-layout.tsx`): a first visit, a returning person and a locked device
  all see the same sign-in page, and it says nothing about what the device
  holds or how it is protected. The security is done, not explained.
- **The account menu** (`src/features/account/user-menu.tsx`): the person's
  picture at the top end of every screen holds every settings page (account,
  sign-in and security, preferences, commitments, categories, payment
  methods, backup), appearance and language switches, and signing out, last.
  Settings is no longer a tab; `/settings` lists the same pages.
- **Profile pictures**: squared and shrunk to 256 px WebP in the browser
  (location and camera details never leave), kept in the private `avatars`
  Storage bucket, one folder per account, readable and writable only by that
  account and only after the second step
  (`supabase/migrations/20261003000000_avatars.sql`). Each device keeps a copy
  for offline. Without one, Google's picture (Google sign-ins) or initials.
- **Linked accounts** (Settings › Account linking, `/settings/linking`;
  `src/features/household/`, `src/db/sync/route.ts`,
  `supabase/migrations/20261007000000_households.sql`): up to ten linked
  accounts that see and edit each other's data. Every account keeps its
  own rows in its own space, sealed with its own key; linking lets the others
  read and write it there (the record functions' `p_space`), and the server
  hands a member the account's own key entry and nothing else of it. A
  transaction, card or account lives in its owner's space (a person's
  `profileId` is their account id); the shared structure (categories,
  the currency, merges) and its joint cards live in every space. Unlinking
  ends the access at once and leaves each account exactly its own - nothing
  is moved or deleted to get there. Invitations go by email (the
  `send-invite` function, through Resend): a single-use link, kept on the
  server only as a hash, good for seven days, accepted only by the signed-in
  account with that confirmed address. Joining asks the invitee to convert
  to the linked accounts' currency first. People made inside an account before
  accounts could link stay listed "from before linking" until their entries
  are given to someone's account; the first of them becomes the account's
  own person. The animal avatars are gone: a person is their account's
  picture, or their initials.
- **Lock after inactivity**, per device: 5 minutes, 15 (the default), an hour,
  a day, or never. The session ends on the server too, not just on screen.
- **Deleting the account** deletes everything it holds, and this device's copy.
- **Delete all data** (Settings › Data & backup): immediate when alone; with linked accounts, every other one confirms by an emailed link first.
- **Two-step sign-in**, optional: a code from an authenticator app after the
  password (or Google). Enforced by the database, not just the screens: every
  function refuses an account with codes turned on unless the session has
  given one (Supabase's `aal2`), and no table can be read directly.
  (`supabase/migrations/20260930000000_second_step.sql`)
- **Passkeys**: sign in with the device's fingerprint, face or screen lock, no
  password. Added and removed in Settings. A passkey is a first factor here; an
  account with codes on still asks for the code after one.
- **Google** hands its answer straight back to bookandry.com, never by way
  of Supabase's own address, so Google's page says "to continue to
  bookandry.com". The answer is a signed ID token, which Supabase checks and
  exchanges for a session; a nonce only the browser that started the sign-in
  holds makes a token lifted from elsewhere useless.
  (`src/services/supabase/id-token.ts`)

What the server holds, per row: a blinded id (an HMAC of table and id), a
revision, a key version, and AES-256-GCM ciphertext of the table, the id, the
row and whether it was deleted. The first device makes the data key and the
account keeps it; every other device fetches it on sign-in.

### Threat model

- **The account is the lock.** Whoever can sign in as the person gets
  everything. Use a passkey, or a strong password with two-step sign-in on.
  Google accounts also have Google's own protections. Losing the
  authenticator app with no passkey means asking the project's operator to
  remove the factor (Authentication → Users in the Supabase dashboard).
- **The server operator, or a breach of the database, can read and change the
  data.** The data key sits in the same database as the ciphertext. The
  encryption keeps rows unreadable to anything that sees only the records
  table (a leaked export of it, a log), and it catches a server replaying a
  row older than one a device has already seen; it is not end-to-end, and it
  does not protect against whoever holds the database.
- **A lost device.** "Sign out all other devices" ends every other session;
  the lost one shows its sign-in screen from then on, and getting past it
  needs the password (and the second factor). Supabase access tokens already
  issued stay valid until they expire, up to an hour. The data on it is not
  encrypted at rest, so someone who takes the storage apart can read it.
- **A shared device.** Signing out removes the data; a different account
  signing in removes the previous one's first. One person's data never reaches
  another's account.
- **Script running in this page** (XSS) can read the local data and use the
  keys while it runs. The Content-Security-Policy (`dist/_headers`, built by
  `src/lib/security-headers.ts`, run under in `e2e/11-csp.spec.ts`) exists to
  stop such a script from running at all. The Supabase session lives in
  `localStorage`, readable by such a script — accepted, for the same reason.

Accounts set up in the two days of passphrase-held keys move over by
themselves, on whichever device signs in first: it makes a new key the account
holds, and every device re-sends what it has.

### Code map

| Where                                            | What                                                              |
| ------------------------------------------------ | ----------------------------------------------------------------- |
| `src/db/sync/middleware.ts`                      | every write records itself for sync, in the same transaction      |
| `src/db/sync/engine.ts`, `apply.ts`, `outbox.ts` | push, pull, last-write-wins merge, repair                         |
| `src/services/crypto/`                           | AES-GCM, HKDF, Argon2id, key wrapping, recovery keys, passkey PRF |
| `src/db/sync/codec.ts`                           | seals and opens records between the engine and the server         |
| `src/db/sync/account.ts`                         | getting a device the account's key; moving it to a new one        |
| `src/app/idle-lock.ts`, `account-gate.tsx`       | locking after inactivity; the sign-in screen in front of the data |
| `supabase/migrations/`                           | tables, row-level security, and the only write path               |
| `src/app/account.ts`, `sync-runner.ts`           | session, key state, when to keep in step                          |
| `src/features/account/`                          | the screens                                                       |

The SQL is tested on real Postgres in-process (PGlite, `src/test/pg-harness.ts`)
— rules, grants and row-level security, and the full two-device path from
Dexie through the codec to the SQL. No Docker needed.
