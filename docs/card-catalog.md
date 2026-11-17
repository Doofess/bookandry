# Card catalog

How the built-in card catalog works.

## The card catalog

Linking a card (Cards, Add a card, "Who issues this card") fills in its real
rules and rates from a catalog: you pick the bank, the card company and the card
type, and the card takes that combination’s rates, its deal types and their
limits. The catalog is a separate chunk, loaded only when a card form or the
purchase form needs it, so it costs nothing at startup. It lives in
`src/data/card-catalog/`.

**What it covers.** Five card companies: Isracard, Amex, Cal, Diners and Max.
Twelve banks that issue them: Hapoalim, Leumi, Mizrahi-Tefahot, Discount,
Mercantile, Massad, Yahav, One Zero, and the FIBI group (First International
(FIBI), PAGI, Otsar HaHayal and U-Bank). Each bank offers the companies whose
cards it issues, so the same company can carry different rates through
different banks.

- **The figures are the published ones, and often maximums.** Banks print the
  most a customer can be charged, not what they will be. A rate is tagged
  "Up to" when it is a maximum, and every rate keeps its source link and the
  date it was read; the info button on a row shows both.
- **An unpublished rate is typed from the statement.** Where nobody publishes a
  figure (Mizrahi’s Credit Plus, for one), the row asks for it, and a purchase
  under that deal will not save until it is there. A published rate can also be
  changed on the card, marked "Changed by you", and put back with one tap.
- **מתואמת is computed, not copied.** The app stores the nominal rate and how
  often it compounds, and works the effective annual rate from those, so it
  follows prime when prime moves. 15.50% compounded monthly shows as 16.65%.
- **Max’s long credit re-prices at payment 19.** Payments 1–18 stay at the rate
  the deal was priced at; from payment 19 the rate moves by any change in prime
  since the purchase. The purchase form shows both halves of the schedule.
- **Split into payments.** A purchase already saved on a linked card can be
  turned into a payment plan in place, but only where the card’s company offers
  it, and on its own terms: each card’s rule says whether it is available, the
  payments allowed, and when interest starts. On Max, interest runs from the day
  you ask; on Isracard (and Amex, which shares its agreement), from the day the
  account is credited, within two business days, or from the regular billing
  date if the purchase has not been charged yet.

- **Plans are edited one payment at a time, or whole.** A payment plan
  (installments, קרדיט, long credit, a loan or a mortgage) no longer offers
  "this and future payments". Splitting a plan there restarted its payment
  count, which also moved the payment-19 re-price. To change every payment
  from now on, edit the whole plan.

**Not covered yet.**

- Revolving credit, ישראקרדיט-style pooled credit, and minimum payments.
  They come later.
- Bank card fees. The catalog does not know them, so the app does not fill them in.
- A scheduled re-check of the published tables. They are read by hand, and each
  rate shows the date it was last read.

The privacy page needs no change: the catalog ships inside the app, and nothing
about a card is sent anywhere to look it up.
