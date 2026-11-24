# CottageMargin

A recipe costing & profit-protection tool for bakers who sell: cottage
bakers supplying cafés, weekly farmers-market vendors, and micro-bakeries
running 10–30 products and buying ingredients in bulk. Tagline:
**"Stop underpricing your baking."**

> **Name note:** this tool was originally built as "BatchBake Ledger" and
> was renamed **CottageMargin** before launch, after name checks found
> collisions for the alternatives (a live "BakeMargin" SaaS and app, an
> existing "CrumbMargin" product, an existing bakery using the BatchBake
> name, and "BakeLedger" software). "CottageMargin" had no direct product
> using it in web searches on Nov 24, 2026 — that is a search check,
> **not a formal trademark clearance**.

Pricing: **$12/month founding-member pricing**, locked in while the
subscription stays active. There is no higher standard tier. (Paid rivals
in this niche cluster at $9–$15/mo; the launch debate rated the original
higher price point at 6.8/10 — indefensible for this audience — and $12
as the price that fits
both the market and the "always affordable" rule.) The calculator, a
watermarked price-sheet preview, and the first 3 saved recipes are free;
Pro unlocks clean PDFs, unlimited recipes, the Ingredient Price Impact
Monitor, and backup export.

Plain static HTML/CSS/JS — no build step, no backend. Local files only:
nothing is deployed from this folder by the build process.

## Files

| File | What it is |
|---|---|
| `index.html` | Landing page: promise, how it works, free/Pro features, a static sample retail price sheet with the baker's private numbers, founding pricing, privacy + billing/cancellation notes. |
| `app.html` | The tool: ingredient library with per-ingredient density conversions, recipe builder with the full cost model (packaging, energy, booth, delivery, labor, waste, sell-through, payment fees), wholesale guard, batch scaling, private cost breakdown, watermarked preview, Pro PDF price sheets (retail and wholesale, jsPDF via CDN), recipe library + dashboard, Ingredient Price Impact Monitor, backup export/import + CSV export. |
| `README.md` | This file. |

## Before launch

- The Stripe placeholder is the literal text `STRIPE_LINK`, and it appears
  **only in quoted-string form**: `href="STRIPE_LINK"` in `index.html`
  (two buttons) and in `app.html` (the subscribe button), plus
  `var STRIPE_URL = "STRIPE_LINK";` in `app.html`. Replace the quoted
  placeholder with the real Stripe subscription payment link for
  CottageMargin ($12/month). Never let the placeholder appear as a
  bare JS identifier — a global find/replace on the bare word would
  corrupt the script.
- Contact email in the footer is `chrisla122787@gmail.com`.

## Free vs Pro

**Free (no payment, no account):**

- Full recipe cost calculator — ingredient library, every input
  (including sell-through, payment fees, energy/booth/delivery), live
  batch math, batch scaling, and thin-margin / unit-mismatch /
  wholesale-below-cost warnings.
- Watermarked price-sheet preview rendered on screen (diagonal
  "PREVIEW — CottageMargin" watermark), in retail or wholesale mode.
- **Save up to 3 recipes** in the recipe library, with dashboard totals.
- **One complete clean PDF price sheet free** (see below).

**Pro (subscription):**

- Unlimited clean branded PDF price sheets — **retail and wholesale
  sheets are separate documents**: the retail sheet shows retail prices
  only, the wholesale sheet shows wholesale prices only, and neither
  ever shows costs, margins, or the labor rate.
- Unlimited saved recipes + dashboard (recipe count, estimated recipe
  margin at set prices, load / duplicate / delete).
- **Ingredient Price Impact Monitor** (see below).
- Backup export (JSON) / import-restore, and CSV export of recipes.

## First sheet free

Free users can generate exactly **one** complete branded PDF:

- A `localStorage` counter (`cottagemargin.freepdf.v1`) tracks how many
  free PDFs have been generated.
- The counter is incremented **only after a PDF is successfully
  generated** (`doc.save()` has run). If the jsPDF library failed to
  load, the recipe numbers don't produce a valid price, or a wholesale
  sheet is requested with no wholesale price set, the attempt is
  rejected with a message and the free PDF is **not** consumed.
- Once the counter reaches 1, further PDF downloads require Pro. Pro
  users generate unlimited PDFs and the counter is never touched.

## ⚠️ MVP access note (read before launch)

Pro unlocking is **client-side only** for launch validation:

- The "I already subscribed — unlock Pro" button in `app.html` simply
  sets a `localStorage` flag (`cottagemargin.pro.v1 = "1"`) in the
  visitor's own browser. There is **no real payment verification** — no
  server checks the Stripe subscription, and anyone who finds the button
  can unlock Pro without paying.
- This is a deliberate MVP trade-off to validate demand cheaply on a $0
  static stack (the landing page says so in plain words: unlocking is on
  the honor system during opening validation).
- Before scaling past validation, replace this with a real entitlement
  check (e.g. a small serverless function that verifies the Stripe
  subscription by email before issuing an unlock token).

## The costing formula

All math is in `calculateRecipe()` in `app.html`. Each recipe line's cost
comes from its ingredient snapshot (see "Snapshot rule" below):

```
lineCost(lib line)  = convertedQty(qty, unit -> ingredient base family) × unitCostPerBase
lineCost(one-off)   = the $ cost typed for that line

ingredientSubtotal  = Σ lineCost
wasteAmount         = ingredientSubtotal × waste% / 100     ← waste applies to ingredients ONLY
laborCost           = laborMinutes / 60 × laborRate
batchCost           = ingredientSubtotal + wasteAmount + packaging + laborCost
                      + other + energy + booth + delivery
costPerItemMade     = batchCost / yieldCount
soldCount           = yieldCount × sellThrough% / 100
costPerSoldItem (C) = batchCost / soldCount
```

**Sell-through** is the share of the batch the baker actually expects to
sell. The items that come home unsold still cost money to make, so the
ones that sold have to carry them: at 90% sell-through, cost per sold
item is the batch cost divided by 90% of the yield — higher than the
cost per item made. All pricing below runs on cost per sold item.

**Pricing, with payment fees.** With target margin M (as a fraction),
payment-fee percentage R, fixed per-item fee F, and cost per sold
item C:

```
plainPrice        = C / (1 − M)                       ← before payment fees
sellingPrice (P)  = (C + F) / (1 − M − R)             ← fee-inclusive price
```

The fee-inclusive price is the headline selling price: it is built so
that after the processor takes R% + F, what remains still delivers the
target margin on the sold item. If **M + R ≥ 100%** no price can do
that — the calculation is rejected, the price shows as "—", and a
plain-words fee error appears. With no fees entered (R = 0, F = 0) the
two prices are identical. Per-item prices are rounded to whole cents,
and margins/profits are then figured from the rounded price — so the
numbers a buyer sees are the numbers the margin is checked against.

```
trueMargin       = (P − P×R − F − C) / P
profitPerItem    = P − P×R − F − C
profitPerBatch   = profitPerItem × soldCount
priceTable[m]    = (C + F) / (1 − m − R)   for m in 30/40/50/60%
```

Why divide by `(1 − margin)` instead of just adding the margin on top of
cost: a profit margin is a share of the final price, not a markup on
cost. Pricing this way means the margin the baker picks (say 50%) is the
share of the selling price that is actually profit. Marking cost up by
50% would only deliver a 33% true margin. A target margin of 100% or
more is rejected as unpriceable.

**Wholesale.** The wholesale price is a chosen discount off the retail
price: `wholesale = round2(retail × (1 − discount%))`. The **wholesale
guard** then checks it against reality:

```
wholesaleProfit  = wholesale − wholesale×R − F − C
wholesaleMargin  = wholesaleProfit / wholesale
minimumWholesale = (C + F) / (1 − minWholesaleMargin − R)
```

If the wholesale profit per item is zero or negative, a plain-words
warning appears ("at this wholesale price you make nothing — or lose
money — on every item sold"). The panel also shows the lowest wholesale
price that still keeps the baker's chosen minimum wholesale margin
(default 20%).

Older saved recipes that predate the newer fields (energy, booth,
delivery, sell-through, fees, minimum wholesale margin) calculate with
safe defaults — sell-through 100%, fees 0, minimum wholesale margin
20% — so their numbers don't change.

## Unit conversions & density

Every ingredient is bought in some unit and used in possibly another.
Costs are kept per **base unit**: grams for weight, ml for volume, each
for count.

| Unit | Family | Base units per unit |
|---|---|---|
| g | weight | 1 g |
| kg | weight | 1,000 g |
| oz | weight | 28.3495 g |
| lb | weight | 453.592 g |
| ml | volume | 1 ml |
| L | volume | 1,000 ml |
| tsp | volume | 4.92892 ml |
| tbsp | volume | 14.7868 ml |
| cup | volume | 236.588 ml |
| each | count | 1 each |
| dozen | count | 12 each |

Same-family conversions use the table directly (2 lb of flour =
907.184 g; 2 dozen eggs = 24 each).

**Weight ↔ volume** (a recipe in cups, flour bought by the pound) goes
through the ingredient's **own grams-per-cup value, stored per
ingredient**: volume → grams = cups × gramsPerCup; grams → volume =
grams / gramsPerCup × 236.588 ml. A cup of flour does not weigh what a
cup of sugar weighs, so the value lives on each ingredient, not on the
app. Editable defaults offered in the library form:

| Ingredient | Grams per cup |
|---|---|
| All-purpose flour | 120 |
| Granulated sugar | 200 |
| Butter | 227 |
| Cocoa powder | 100 |
| Honey | 340 |

120 (the flour convention) is only the fallback when an ingredient has
no value set.

**Count ↔ weight/volume cannot be converted** (there is no honest way to
turn "3 eggs" into grams without an egg weight). Such a line is counted
as $0, flagged with a "unit mismatch" warning, and excluded from the
ingredient subtotal until the unit is fixed.

An ingredient's cost per base unit is
`purchasePrice / (purchaseQty × toBase(purchaseUnit))` — e.g. a 25 lb bag
at $18.50 costs $18.50 / 11,339.8 g ≈ $0.00163 per gram.

## Snapshot rule (important)

**A recipe line keeps the ingredient price from the moment it was
picked.** When a library ingredient is chosen in a line, the line stores
its own snapshot: ingredient name, base family, cost per base unit,
grams per cup, and the purchase qty/unit/price it came from. After that:

- Editing the ingredient's price in the library does **not** change any
  existing line — in the recipe being built or in any saved recipe.
- **Re-picking** the ingredient in the line (choosing it again in the
  line's ingredient dropdown) refreshes that line to the library's
  current price.
- Deleting the ingredient from the library leaves lines untouched; the
  line's dropdown shows the name with "(saved price)".
- One-off lines store the typed name, quantity, unit, and $ cost
  directly.

This is deliberate: a recipe costed last month must not silently change
because flour went up — the baker decides when to re-cost. The Price
Impact Monitor (below) is the controlled way to apply a price change.

## Ingredient Price Impact Monitor (Pro)

The retention feature. When a library ingredient's **current** purchase
price differs from the price snapshotted into a saved recipe, the
monitor lists that recipe as affected — one row per recipe + ingredient:

- recipe name and ingredient name;
- old → new purchase price, with the % change;
- the recipe's margin **at its current set price** under the old cost
  vs under the new cost (the new cost is computed hypothetically by
  re-costing a copy of the recipe with the library's current price for
  that one ingredient — **snapshots are not modified**);
- the recommended new retail price that restores the recipe's target
  margin under the new cost (fee-inclusive, same formula as above).

Each row has an **"Apply new price"** button. Applying re-snapshots
**only that ingredient, only in that recipe**, at the current library
price, recomputes, and saves. Every other ingredient's snapshot and
every other recipe is untouched, and the impact row disappears. The
recipe currently being edited (unsaved) is never listed — the monitor
covers the saved library only.

## Batch scaling

Scaling is **linear**: pick a multiplier (×2.5) or a target item count
(factor = target / yield), and every quantity and every cost multiplies
by the same factor. Cost per item is therefore unchanged by scaling.

Displayed scaled quantities are rounded for the kitchen — **weights to
whole grams, volumes and counts to the nearest 0.25** — but that rounding
is display-only: costs are always figured from the exact quantities. The
price sheets and preview show the scaled batch (default ×1, the recipe
as written).

## Worked example — the pre-loaded sample

The app opens with a sample recipe (and a matching sample ingredient
library) so the math can be checked end to end. Vanilla Bean Cupcakes,
yield 12, sell-through 100%, no payment fees, no energy/booth/delivery:

| Line | Math | Cost |
|---|---|---|
| All-purpose flour, 240 g | 240 × ($6.40 / 4,535.92 g) | $0.3386 |
| Granulated sugar, 200 g | 200 × ($3.84 / 1,814.368 g) | $0.4233 |
| Unsalted butter, 170 g | 170 × ($4.48 / 453.592 g) | $1.6790 |
| Eggs, 3 each | 3 × ($3.60 / 12) | $0.9000 |
| Vanilla extract, 10 ml | 10 × ($8.30 / 100 ml) | $0.8300 |
| Whole milk, 120 ml | 120 × ($1.55 / 1,000 ml) | $0.1860 |

- Ingredient subtotal: **$4.3570**
- Waste 5% (ingredients only): **$0.2179** → ingredients with waste **$4.5749**
- Packaging $2.40 · Labor 45 min × $18/hr = $13.50 · Other $0.60 ·
  Energy $0 · Booth $0 · Delivery $0
- **Batch cost: $21.0749 → $21.07** · Cost per cupcake made: **$1.7562 → $1.76**
- Cost per cupcake sold (100% sell-through): **$1.76**
- Price at 50% margin: $1.7562 / 0.50 = **$3.51** (true margin at $3.51: 49.96%)
- Wholesale at 20% off: $3.51 × 0.80 = **$2.81** · wholesale profit per
  item $1.05 · wholesale margin 37.5%
- Lowest wholesale price at a 20% minimum margin: **$2.20**
- Full batch at retail: 12 × $3.51 = **$42.12** · Profit per batch: **$21.05**
- Price table: 30% → $2.51 · 40% → $2.93 · 50% → $3.51 · 60% → $4.39
- Scaled ×2.5: 30 cupcakes, batch cost $52.69, flour 600 g, eggs 7.5 each

The landing page's sample sheet and private-numbers box show exactly
these figures; the app, landing, and this README agree to the cent.
(With sell-through below 100% or payment fees entered, the app's
suggested price rises accordingly — e.g. at 90% sell-through the same
recipe prices at $3.90, and with a 10% + $0.30 payment fee at $5.14.)

## Data & privacy

Everything lives only in the visitor's own browser (`localStorage`):

| Key | Contents |
|---|---|
| `cottagemargin.ingredients.v1` | Ingredient library (name, purchase qty/unit/price, grams per cup). Seeded with the 6 sample ingredients on first run only. |
| `cottagemargin.recipes.v1` | Saved recipes (scalar inputs + ingredient line snapshots, name, date). |
| `cottagemargin.settings.v1` | Bakery name/phone/email + logo (data URL). |
| `cottagemargin.pro.v1` | Pro unlock flag (see MVP note above). |
| `cottagemargin.freepdf.v1` | Count of free PDFs generated (first sheet free). |

Nothing is sent to a server. The app tells users plainly: "Your data is
stored only in this browser on this device — export a backup regularly."
Backup export downloads a JSON file with settings + ingredients +
recipes; import restores it (files from other tools — including backups
from this tool's pre-rename version — are rejected by the
`app: "CottageMargin"` marker). CSV export lists saved recipes with
their computed costs (per item made and per item sold), sell-through,
and prices.

## Deploy

Any static host works — copy the three files (or at least the two HTML
files) to the host as-is:

- **Netlify / Vercel:** drag-and-drop the folder, or connect the repo; no
  build command, publish directory = this folder.
- **Cloudflare Pages:** same — framework preset "None", no build command.
- **GitHub Pages:** fine for demos/preview only — GitHub Pages terms
  prohibit primarily commercial SaaS use, so don't run the paid product
  from there.

The only external request the app makes is the jsPDF CDN script in
`app.html` (cdnjs). If it fails to load, everything except the PDF button
still works and the user gets a clear message.
