# Deploy

The Supabase project configuration that lives in the dashboard rather than in this repository.

## Supabase project checklist

Configuration that lives in the dashboard, not in this repo:

- Run `supabase/migrations/*.sql` (SQL editor, or `supabase db push`).
- **Authentication → Providers → Email**: confirm email ON; secure password
  change ON; leaked-password protection ON (Pro plan).
- **Authentication → Providers → Google**: client id and secret — the secret
  lives only here, never in this repo or `.env.local`. Skip nonce check OFF.
- **Google Cloud → the OAuth web client → Authorized redirect URIs**: add
  `https://bookandry.com/auth/callback` (keep the supabase.co callback, for
  builds without `VITE_GOOGLE_CLIENT_ID`). **Branding**: app name
  Bookandry; home page `https://bookandry.com`; privacy policy
  `https://bookandry.com/privacy`; terms `https://bookandry.com/terms`
  (`public/privacy.html`, `public/terms.html`: plain pages, open without
  signing in); authorized domain `bookandry.com`, verified in Search Console.
  Verifying the brand shows the name and logo too.
- **Authentication → Multi-Factor**: App Authenticator (TOTP) enabled (the
  default).
- **Authentication → Passkeys**: enabled; relying party ID `bookandry.com`,
  display name `Bookandry`, origin `https://bookandry.com`. The ID is
  permanent in practice: changing it strands every passkey already made.
- **Authentication → Emails → SMTP Settings**: custom SMTP on (Resend:
  `smtp.resend.com`, port 465, user `resend`, the API key as the password —
  entered here only), sender `no-reply@bookandry.com`, name `Bookandry`.
  Without it Supabase sends as "Supabase Auth", and **only to members of the
  project's team**: nobody else gets a confirmation or reset email.
- **Authentication → Emails → Templates**: "Confirm sign up" and "Reset
  password" take the subject and body from `supabase/templates/`
  (`pnpm emails` rebuilds them and `public/email/`; deploy before pasting, as
  the emails load their images from the site). Each email comes in the
  account's language, kept in `user_metadata.language`.
- **Edge Functions → Secrets**: `ANTHROPIC_API_KEY`, for `categorize`
  (`supabase/functions/categorize/`), which picks a category from what was
  typed ("Claude Max Plan" → AI & software). Entered here only. Without it
  the function answers 503 and the person picks the category, as before.
  Optional `CATEGORIZE_MODEL` overrides the model (Claude Haiku 4.5).
- **Edge Function `extract-transactions`** (`supabase/functions/extract-transactions/`,
  with `_shared/import-wire.ts`, a copy of `src/domain/import-wire.ts` that a
  test keeps identical): "Smart import". Same `ANTHROPIC_API_KEY`. Optional
  `EXTRACT_MODEL` (Claude Sonnet 5.5), `CHECK_MODEL` (Claude Haiku 4.5, the
  second read) and `VERIFY_MODEL` (Claude Opus 5.5, the correction), each
  only from an allow-list. Its daily cap is the migration
  `20261004000100_ai_usage_one_cap.sql` (80 calls a person a day).
- **Edge Function `send-invite`** (`supabase/functions/send-invite/`): emails
  an invitation to link accounts. Needs **Edge Functions → Secrets →
  `RESEND_API_KEY`** (the same Resend key as the SMTP password; entered there
  only). Without it, inviting answers "the email couldn't be sent" and no
  invitation is left behind.
- **SheetJS** is pinned to its own CDN's tarball
  (`https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz`); the npm `xlsx`
  package is years behind and unmaintained. `pnpm outdated` does not track it.
- **Edge Function `prime-rate`** (`supabase/functions/prime-rate/`), deployed
  with `--no-verify-jwt`: public data, no secrets. It relays the Bank of
  Israel's rate (its public API, with the SDMX statistics series as a
  fallback), because the Bank's API sends no CORS headers.
- **Authentication → URL Configuration**: Site URL `https://bookandry.com`;
  redirect URLs `https://bookandry.com/settings` (plus a localhost one for
  development only).
- `.env.local` holds only public values: `VITE_SUPABASE_URL`,
  `VITE_SUPABASE_PUBLISHABLE_KEY` and `VITE_GOOGLE_CLIENT_ID`. Never the service-role key, and never the `client_secret_*.json` Google
  lets you download.
