# Known limitations

What the app does not do, or does not do well, today.

- "Smart import" from the Share menu is Android's, for the installed app, and
  needs the service worker installed (open the app once); iPhone has no
  share target, so there it is the file picker. Its accuracy has been checked
  on the checks' own test cases, not yet measured on a set of real
  statements and notifications.
- Release note: Max and Discount spreadsheets imported before the exact
  readers (`import-sheet-readers.ts`) may have been saved with wrong
  amounts - a whole deal or loan on each payment, a foreign figure as
  shekels. Re-importing them will not match those rows and will add them
  again, so delete the old ones first.
- Instalment plans are read-only in the editor. Re-splitting an existing plan
  would break the sum-to-total invariant.
- Built-in categories can be archived but not renamed or deleted: they store an
  i18n key, so a rename would have to fork them into custom rows.
- Turning reduced-motion **off** mid-session leaves the last animated figure on
  screen until the value next changes.
- Flags are rendered as code badges, not emoji. Windows ships no flag glyphs.
