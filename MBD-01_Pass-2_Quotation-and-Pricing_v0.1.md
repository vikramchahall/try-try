# MBD-01 Pass 2 — Quotation and Pricing Build Document

**Module:** M2-B Quotation · **Prefix:** REQ-QT · **Tier:** 3
**Version:** v0.1 (draft for review)
**Date:** 6 October 2026
**Source:** PRD v0.5 §7.2.1 (REQ-QT-01 to 04) · M2 Leads Management and Quotation v0.2 §4, §6, §8, §21 ·
`B2B Price list Sep 2026.xlsx` · `ZBOTREE_CONSL 150926.xlsx` (the SAP product tree behind
`db/data/product-master.csv`) · *Volume two — Quotation on the phone* (mobile wireframes, MQT-01 to 11) ·
*Quotation — App screen spec and build process* (16-Sep-2026)
**Parent:** BATD v0.4 · MBD-01 v0.4 (pass 1) · PRD v0.5
**Folds into:** MBD-01 v1.0, which is pass 1 and this document together (D-01)
**Status:** DRAFT — not yet approved for development

---

> **Read this first.** Two things shape everything below.
>
> 1. **Quotation is two modules, not one.** `pricing` owns the price list and answers "what is the
>    list price of this SKU, at this quantity, today". `quotations` owns the quote and never reads a
>    price table. The parts blocked by business decisions (OD-3, the six price list questions) are
>    therefore inside `pricing`, behind one port, and the quote can be built today against it.
> 2. **Every open business rule is configuration or a swappable policy, never a constant.** Bands,
>    triggers, tax mode, quantity-band basis, the stage at which quoting opens, the number format —
>    each is a config key with `pendingDecision`, or a pure policy with a stub. When the business
>    answers, the change is a config write or one file, not a migration.
>
> **An external "quotation specification" is cited by the code and is not in this repository.**
> Comments in `masters/ports.ts`, `0025`–`0027` and `13_terms_provisional.ts` refer to its §1.1,
> §3.4, §4.2, §7.3 and §10.2, and root `PENDING.md` §5a refers to a five-wave "delivery plan" and
> "OD-4c, five triggers". Neither document was available when this was written. Where this
> document reconstructs those rules from the code comments it says so. **Whoever holds that
> specification should check §6 and §8 against it before this is approved.**

---

## 0. What this document decides

| | Decision | Why, in one line |
|---|---|---|
| 1 | Two modules: `pricing` and `quotations` (§1.3) | The blocked half and the buildable half stop sharing a migration |
| 2 | A quote is a **chain** (`quotation`) of **versions** (`quotation_version`) (§3.2) | The number belongs to the chain, the approval and PDF to the version — the wireframe's `…/00041 · v2` |
| 3 | `place_of_supply_state_code` is an FK to `state(code)` — `IN-TN`, not `char(2)` (§3.2) | Settles root `PENDING.md` §4.2a in the migration that creates the column, as it asks |
| 4 | Price list items are exploded **per SKU** at import, through a reusable alias table (§3.1) | The sheet is keyed by a size-level name; a quote line is a material. One curation, not a string join |
| 5 | Approval bands are a **config key**, not a table (§5.3) | BATD defines them twice; config already has history, audit and a portal editor |
| 6 | `EXPIRED` is **derived**, not stored (§3.2) | No job has to run for a quote to be expired, and none can forget to |
| 7 | A rejection returns the **same version** to `DRAFT` (§8) | MQT-08: "Editing keeps the same quotation." M2 §6.2 said a new version; the drawing is the later decision |
| 8 | Issue is synchronous; the **PDF is not** (§8.4) | Allocating a number cannot wait on headless Chromium |
| 9 | Tax is a **policy interface with a stub** until OD-4a (§5.4) | Root `PENDING.md` §5a's own suggestion |
| 10 | Quotations never writes to `opportunity` except through one write port (§1.4) | The stage advance on issue; enquiry value is never touched (REQ-LM-04) |

---

## 1. Scope and placement

### 1.1 In scope

REQ-QT-01 to 04: the B2B price list (upload, validate, publish, supersede, resolve), quote build
against an Opportunity, variance, the approval engine and queue, issue with an immutable PDF,
revision and versioning, re-pricing a stale draft, and the stage advance to Quote Submitted.

### 1.2 Out of scope

- **Asking for a new payment or delivery term (MQT-11).** That is a request on a `masters` entity
  and belongs to the masters request flow, not here. Noted in §15.
- **The approval matrix editor screen.** The matrix is a config key (§5.3); the existing config
  screen edits it.
- **Email and SMS delivery** (gap G-6). This module emits events; in-app delivery is a platform
  prerequisite (§2, P-5).
- **Order booking, SAP push, MRP.** The second SAP extract has not arrived; nothing here reads
  `sku_mrp_history`.

### 1.3 Two modules, and why

```
                 ┌──────────────────────────┐        ┌──────────────────────────┐
                 │  pricing                 │        │  quotations              │
                 │  price_list              │ price  │  quotation               │
   Admin ──────▶ │  price_list_item         │ Read   │  quotation_version       │ ◀── Salesperson
   upload        │  price_source_alias      │ ─────▶ │  quotation_line          │
                 │  publishes: priceRead    │        │  quotation_approval      │
                 └──────────────────────────┘        └───────────┬──────────────┘
                                                                  │ reads through ports
                   masters: skuRead · customerRead · termsRead ───┤
                   opportunities: opportunityRead · opportunityStage ┤ (both new, §2 P-1)
                   contacts: contactRead ─────────────────────────┘
```

**`pricing` — `src/modules/pricing/`.** Owns the three price tables and publishes one read port.
It knows nothing about quotes. PRD §3.2 puts the price list in Layer L, and MBD-06 §3.0 already
established that Layer L is not a place in this codebase; it is a set of ordinary modules. Pricing
is not folded into `masters` because it has a lifecycle masters does not (draft → validated →
scheduled → live → superseded), its own job, and — decisively — it is the part blocked on OD-3.
Keeping it separate means the blocked half can change shape without touching `masters` or
`quotations`.

**`quotations` — `src/modules/quotations/`.** Owns the quote and the approval decision. Asks
`pricing` for a list price and never reads `price_list_item`. Asks `masters` for SKU, customer and
term data and snapshots what it needs. Asks `opportunities` for the opportunity and, on issue, to
advance its stage.

**Why no separate `approvals` module.** BATD §3.7 sketched `approval_request` with an
`entity_type`, anticipating documents (Tier 5) and warranty (Tier 7). Building a generic engine for
one consumer is the speculative generality this repository has avoided elsewhere. The decision
record is `quotation_approval`; the evaluation is a pure domain component (§5.2) with no
quotation-specific I/O. If Tier 5 needs it, it is lifted out then — the shape is already
entity-agnostic.

### 1.4 Coupling rules this module holds to

- **`no-cross-module-import` from the first file.** Each consumer declares its own copy of every
  port interface and resolves it with `deps.ports.get<T>(name)` — the `CalendarSource` precedent
  in `attendance/application/expected-days.ts`.
- **Foreign keys across modules are allowed; queries across modules are not.** `quotation_version`
  carries `price_list_id REFERENCES price_list(id)` and `sku_code REFERENCES sku(code)` for
  integrity, but `quotations` never joins into `price_list_item` or `sku`. It reads through ports.
  (BATD §2.3: cross-module access is "by read port, event or foreign key".)
- **Atomic cross-module writes use a write port in the caller's transaction**, not an event — the
  `opportunityCreation` precedent. There is exactly one: the stage advance on issue.
- **Everything after the transaction is an event.** Notifications, the PDF render, the ageing
  alert. Seven quote events are already declared in `platform/events/event.ts`.
- **No stage literal.** The target stage is a config key (`quotation.issue.advanceToStage`), because
  `no-stage-literals` forbids `'QUOTE_SUBMITTED'` outside the seed and `opportunities/config.ts`.

---

## 2. Prerequisites

| # | Prerequisite | Owner | State |
|---|---|---|---|
| P-1 | **`opportunityRead.getForQuoting` and a stage write port** in `opportunities` | opportunities | **Not built.** Promised by MBD-01 §2.10, recorded missing in root `PENDING.md` §4.1. Its own small branch, first. Shapes in §4.2 |
| P-2 | **Site-level customer read** — `customerRead.getSiteForQuoting` | masters | Not built. D-23 and CR-04 §5 make the site's GSTIN and address operative on the quote; today's `getForQuoting` returns the organisation's |
| P-3 | **Widen `termsRead.getDeliveryTerm`** to return `freightBy`, `insuranceBy`, `unloadingBy`, `installationIncluded`, `partialDispatchAllowed` | masters | Not built. The quote snapshots the term whole (REQ-QT-04); today's port omits five columns |
| P-4 | **PDF renderer** in `src/platform/pdf/` — `render(templateKey, data) → Buffer` | platform | Empty directory. Generic, no domain logic. Headless Chromium in the worker (AD-11) |
| P-5 | **In-app notification writer** — a subscriber that turns events into `notification` rows | platform | `src/platform/notifications/` is empty. The approval queue is the first thing that needs a recipient (BATD) |
| P-6 | **Price list file and alias list committed** to `backend/db/data/` | this build | The Sep 2026 sheet is still in someone's Downloads folder (root `PENDING.md` §1.4) |
| P-7 | **Branch base** | — | Build on `main` after `fix/attendance-storage` merges, so migrations continue from `0039` |

P-1 to P-3 are small and unblocked. P-4 and P-5 are platform work this module is the first to need.

---

## 3. Data model

Every table **unqualified** (P31). `id uuid PRIMARY KEY` with no default — the application
generates UUIDv7. Money `numeric(18,4)` in rupees (AD-12). Variance percentages `numeric(7,4)` —
a markup above list can exceed 100% and a band edge needs four decimals; tax rates stay
`numeric(5,2)` as in `hsn_rate`. Every table ends with `attach_audit(current_schema(), '<table>')`.

### 3.1 `pricing` tables

#### `price_list`

One uploaded sheet, versioned.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | pk |
| `version_no` | integer | no | `UNIQUE`. Global sequence — the wireframes say "price list v4", not "B2B v4" |
| `name` | text | no | "B2B Pricing — September 2026" |
| `segment` | text | no | `CHECK IN ('B2B','B2G','ALL')`. Q-5 below; the Sep sheet is `B2B` |
| `customer_category_id` | uuid | yes | FK `master_value(id)`. Null = all categories. BQ-47 — empty today |
| `zone_id` | uuid | yes | FK `zone(id)`. Null = national. No sheet has used it |
| `effective_from` | date | no | |
| `effective_to` | date | yes | Set when superseded. Null on the live list |
| `status` | text | no | `CHECK IN ('DRAFT','VALIDATED','SCHEDULED','LIVE','SUPERSEDED','WITHDRAWN')` |
| `currency` | char(3) | no | `DEFAULT 'INR'` |
| `tax_basis` | text | no | `CHECK IN ('EXCLUSIVE_OF_GST','INCLUSIVE_OF_GST')`. Sep sheet: exclusive |
| `price_basis` | text | no | `CHECK IN ('PRICE_INCLUSIVE_OF_FREIGHT','PRICE_EXCLUSIVE','UNKNOWN')` — the vocabulary of `delivery_term.price_basis`, plus `UNKNOWN`. OD-3a. **`UNKNOWN` until answered, and §5.2 treats it as a mismatch, never as a default** |
| `always_requires_approval` | boolean | no | `DEFAULT false`. Note 3 of the Sep sheet ("approval required for every B2B order"). Q-6 |
| `printed_notes` | text[] | no | `DEFAULT '{}'`. The sheet's policy notes, in order, printed on the quote if config says so |
| `source_attachment_id` | uuid | yes | FK `attachment(id)`. The uploaded file, kept |
| `source_checksum` | text | yes | SHA-256 of the upload |
| `validation_report` | jsonb | yes | Written by validate (§8.1). Blocking issues, unmatched labels, diff against the live list |
| `published_by` / `published_at` | uuid / timestamptz | yes | |
| trailer | | | `created_at`, `created_by`, `updated_at`, `updated_by`, `version` |

```sql
-- no two live lists for the same slice can overlap in time
ALTER TABLE price_list ADD CONSTRAINT price_list_no_overlap EXCLUDE USING gist (
  segment WITH =,
  coalesce(customer_category_id, '00000000-0000-0000-0000-000000000000'::uuid) WITH =,
  coalesce(zone_id,              '00000000-0000-0000-0000-000000000000'::uuid) WITH =,
  daterange(effective_from, effective_to, '[)') WITH &&
) WHERE (status IN ('SCHEDULED','LIVE','SUPERSEDED'));

ALTER TABLE price_list ADD CONSTRAINT price_list_published_complete CHECK (
  status IN ('DRAFT','VALIDATED','WITHDRAWN')
  OR (published_by IS NOT NULL AND published_at IS NOT NULL AND validation_report IS NOT NULL)
);
```

`btree_gist` is installed by `0001`. Dimensions sit on the **header**, not the item: the Sep sheet
has none, and a future category-specific sheet is a second list, not a second column. This is the
shape root `PENDING.md` §1.4 proposed, and it replaces BATD §3.7's per-item dimensions and its
`specificity` column.

#### `price_list_item`

One price per SKU per quantity band. **Exploded per material at import** — ~2,900 rows from the
101-row Sep sheet.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | pk |
| `price_list_id` | uuid | no | FK `price_list(id)` |
| `sku_code` | text | no | FK `sku(code)` — the FG material number |
| `min_qty` | numeric(12,2) | no | Lower edge of the band, inclusive. `CHECK > 0` |
| `unit_price` | numeric(18,4) | no | `CHECK >= 0`. **Stored exact** — the sheet displays 2132 and stores 2131.5 |
| `floor_price` | numeric(18,4) | yes | Hard floor (REQ-QT-03, MQT-07c). Not in the Sep sheet; null = no floor |
| `source_alias_id` | uuid | yes | FK `price_source_alias(id)`. Which curated mapping produced this row |
| `source_label` | text | no | The sheet's text exactly as uploaded, for the validation report |
| `source_row` | integer | yes | Spreadsheet row number |
| `created_at`, `created_by` | | | Append-only: no `version`, no `updated_*` — the 0025 `hsn_rate` precedent |

```sql
CREATE UNIQUE INDEX price_list_item_band_uq ON price_list_item (price_list_id, sku_code, min_qty);
ALTER TABLE price_list_item ADD CONSTRAINT price_list_item_floor_below_price
  CHECK (floor_price IS NULL OR floor_price <= unit_price);
```

**Bands as `min_qty` rows, not three price columns.** Resolve with
`WHERE min_qty <= :qty ORDER BY min_qty DESC LIMIT 1`. A fourth band is four rows, not a migration.
The Sep sheet becomes `1`, `201`, `501` — **which puts quantity 200 in the lowest band**. The sheet
says "less than 200" and "201 to 500", so 200 belongs to neither; this is price list Q-1, and the
importer's band edges are a config key until it is answered (§5.5).

**Why per SKU and not per size node.** The sheet's "Market SKU" is a size-and-finish name
("Aris Contour 1200 Wooden"), matching 0 SAP Market SKU codes and between 1 and 40 materials each.
A quote line is a material (FG code), the SKU picker searches materials, and `sku_code` is the key
the rest of the build already uses. Exploding once at import, under review, keeps the resolver a
single indexed lookup and keeps the FK honest. The cost is ~2,900 rows per list, which is nothing.

**Immutability.** A `price_list_item` row is never updated or deleted once its list leaves `DRAFT`.
Trigger in the `attendance_day` pattern (§4). A correction is a new list version.

#### `price_source_alias`

The reusable mapping from a spreadsheet label to materials. Curated once, reused by every later
upload — this is what makes the next sheet a five-minute import rather than a day of matching.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | pk |
| `source_label_norm` | text | no | `UNIQUE`. Lower-case, NBSP → space, whitespace collapsed. "renesa prime 1200 wood" |
| `product_node_id` | uuid | no | FK `product_node(id)`. Normally the `SIZE` node |
| `colour_codes` | text[] | yes | Restricts to these `COLOUR` children. Null = every colour under the node. Carries "Wooden" = {Dark Teakwood, Walnut, …} |
| `sub_variant` | text | yes | `Smart` / `Non-Smart`, where the label implies it |
| `match_method` | text | no | `CHECK IN ('EXACT','NORMALISED','HEURISTIC','MANUAL')` |
| `is_active` | boolean | no | Deactivate, never delete |
| `approved_by` / `approved_at` | | yes | An alias is used by an import only once approved |
| trailer | | | |

Analysis of the Sep sheet against the 1,068-row product tree: 17 labels match a size name exactly,
33 after stripping finish words, **89 of 101** with six word-order heuristics (`Reg` → Regulator
model, `Exhaust X n` → `X Exhaust n`, `Pedestal`/`Wall` reordering, drop `Swing`), and the last
**12** need a manual alias (Efficio Prime Reg → "Efficio Prime RG", Renesa Elite Liquid/Wooden →
"Smart_Metallic/Smart_Wooden", Studio Nexus likewise, Renesa Table Fan → "Renesa Table 400").
That curated list is committed with the file (P-6) as `db/data/price-source-aliases.csv` and loaded
by a seed.

**Do not import the sheet's "SKU Family" column.** Rows 89–101 carry "Renesa Prime" by a pasted-over
VLOOKUP into a workbook that is not supplied. Family comes from `product_node`.

### 3.2 `quotations` tables

#### `quotation` — the chain

What the customer knows as "the quotation": one number, many versions.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | pk |
| `opportunity_id` | uuid | no | FK `opportunity(id)` |
| `quote_no` | text | yes | `UNIQUE`. **Allocated at first issue**, never at draft (A-M2-16) — abandoned drafts consume no number |
| `segment` | text | no | `CHECK IN ('B2B','B2G')`. Snapshot at creation from `organisation.is_government` — the same derivation 0027 uses for `payment_term.applies_to`, **not** `opportunity.category` |
| `owner_user_id` | uuid | no | FK `app_user(id)`. The salesperson; drives write scope |
| `zone_id` | uuid | no | FK `zone(id)`. Snapshot of the opportunity's zone at creation, immutable |
| `status` | text | no | `CHECK IN ('OPEN','CANCELLED')`. The chain is open while any version can still move |
| trailer | | | `created_at`, `created_by NOT NULL`, `updated_at`, `updated_by NOT NULL`, `version` |

`attach_zone_rls(current_schema(), 'quotation', 'owner_user_id')` — inert since P30, written anyway;
`scopePredicate` is the live control.

**Several quotations per opportunity are allowed.** Q-M2-05 is open. A chain table permits both
answers; BATD's `UNIQUE (opportunity_id, version_no)` permitted only "one". If the answer is one,
it becomes a partial unique index on `opportunity_id WHERE status = 'OPEN'` — an index, not a
redesign.

#### `quotation_version` — what is priced, approved, issued

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | pk |
| `quotation_id` | uuid | no | FK `quotation(id)` |
| `version_no` | integer | no | 1, 2, 3 … `UNIQUE (quotation_id, version_no)` |
| `supersedes_version_id` | uuid | yes | FK self. The issued version this revises |
| `status` | text | no | `CHECK IN ('DRAFT','PENDING_APPROVAL','ISSUED','SUPERSEDED','CANCELLED')` (§8) |
| **Pricing basis** | | | |
| `price_list_id` | uuid | no | FK `price_list(id)`. The list this version is priced against (REQ-QT-01 AC) |
| `price_list_version_no` | integer | no | Snapshot, so a read never needs pricing |
| **Addressee — snapshot at creation, refreshed while DRAFT** | | | |
| `organisation_id` | uuid | no | FK `organisation(id)` |
| `site_id` | uuid | no | FK `site(id)` (CR-04 §5) |
| `customer_name` | text | no | |
| `customer_gstin` | text | yes | The **site's** GSTIN, falling back to the organisation's (D-23). Null routes to approval (§5.2, UNREGISTERED_BUYER) |
| `billing_address` | text | yes | From the site |
| `attention_contact_id` | uuid | yes | FK `contact(id)` |
| `attention_name` / `attention_designation` | text | yes | Snapshot — "K. Raghavan · Estate Office" |
| `place_of_supply_state_code` | text | no | **FK `state(code)` — `IN-TN`**. Derived from the site's state; editable only with `quotation:override_tax` (MQT-02, OD-4a). This settles root `PENDING.md` §4.2a |
| `customer_category_id` | uuid | yes | Snapshot; what the price resolver was asked with |
| **Terms — snapshot** | | | |
| `payment_term_code` | varchar(16) | no | FK `payment_term(code)` |
| `payment_term` | jsonb | no | Every column `termsRead` returns, including `printedText`. Frozen at issue |
| `delivery_term_code` | varchar(16) | no | FK `delivery_term(code)` |
| `delivery_term` | jsonb | no | Likewise, after P-3 widens the port |
| `lead_time_days` | integer | no | `CHECK >= 0`. Defaulted from the term, editable (MQT-05) |
| `additional_conditions` | text | yes | Printed verbatim (MQT-06). Non-empty routes to approval |
| `validity_date` | date | no | **Mandatory.** PRD says "recommended mandatory"; BATD left it nullable. A quote without a validity date cannot expire and so never becomes stale in a customer's hands |
| **Totals — written by the server on every line change** | | | |
| `subtotal` | numeric(18,4) | no | Σ line value |
| `list_subtotal` | numeric(18,4) | no | Σ list line value |
| `freight_amount` | numeric(18,4) | no | `DEFAULT 0`. OD-4a. Zero with `freight_mode = 'INCLUDED'` under an inclusive term |
| `freight_mode` | text | no | `CHECK IN ('INCLUDED','CHARGED','EXTRA_AT_ACTUALS','PER_TENDER')` |
| `taxable_value` | numeric(18,4) | no | |
| `cgst_amount` / `sgst_amount` / `igst_amount` | numeric(18,4) | no | |
| `grand_total` | numeric(18,4) | no | |
| `tax_policy` | text | no | Which `TaxPolicy` computed the figures — `STUB_V0` until OD-4a (§5.4). Printed nowhere; recorded so a stub-taxed quote can never be mistaken for a real one |
| `total_variance_pct` | numeric(7,4) | yes | Null when `list_subtotal = 0` |
| `worst_line_variance_pct` | numeric(7,4) | yes | Always stored, whichever basis config picks |
| `evaluation` | jsonb | yes | The last approval evaluation (§5.2) — the list MQT-07 renders |
| **Issue** | | | |
| `issued_at` / `issued_by` | timestamptz / uuid | yes | |
| `pdf_attachment_id` | uuid | yes | FK `attachment(id)`, `owner_type = 'QUOTE_PDF'` |
| `pdf_checksum` | text | yes | SHA-256 (AD-11). Rendered once, never regenerated |
| `pdf_status` | text | yes | `CHECK IN ('PENDING','READY','FAILED')`. Null until issue |
| trailer | | | `version` drives `If-Match` on every draft write |

```sql
CREATE UNIQUE INDEX quotation_version_one_open
  ON quotation_version (quotation_id)
  WHERE status IN ('DRAFT','PENDING_APPROVAL');      -- one version in flight per chain

ALTER TABLE quotation_version ADD CONSTRAINT qv_issued_complete CHECK (
  status NOT IN ('ISSUED','SUPERSEDED')
  OR (issued_at IS NOT NULL AND issued_by IS NOT NULL AND pdf_status IS NOT NULL)
);
ALTER TABLE quotation_version ADD CONSTRAINT qv_conditions_not_blank
  CHECK (additional_conditions IS NULL OR length(btrim(additional_conditions)) > 0);
```

**Why `EXPIRED` is not a status.** It is `status = 'ISSUED' AND validity_date < today (IST)`,
computed on read and returned as `isExpired`. A stored status needs a job to set it, the job runs
nowhere today (MBD-06 §10), and a quote that stays `ISSUED` past its date because a job did not
run is a quote the customer can still act on. BATD's `CANCELLED` is kept, for a draft abandoned;
an issued quote is never cancelled, only superseded (Q-M2-23 stays open — §15).

**Why `is_stale` is not a column either.** A draft is stale when the live list for its slice is no
longer `price_list_id`. That is one port call on read (§4.3), and it cannot drift.

#### `quotation_line`

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | pk |
| `quotation_version_id` | uuid | no | FK `quotation_version(id)` |
| `line_no` | integer | no | `UNIQUE (quotation_version_id, line_no)` |
| `sku_code` | text | no | FK `sku(code)` |
| `description` | text | no | Snapshot of `displayName` — **never typed** (M2 §4.4) |
| `uom` | text | no | Snapshot |
| `hsn_code` | varchar(8) | no | Snapshot |
| `gst_rate_pct` | numeric(5,2) | no | Snapshot of the rate in force, re-read at issue. `0` is a rate; absent is a refusal |
| `quantity` | numeric(12,2) | no | `CHECK > 0`. Matches `sku.moq`; OD-8 fractional ATP keeps it decimal |
| `price_list_item_id` | uuid | yes | FK `price_list_item(id)`. The row the resolver chose. Null only for a manual price (Q-3) |
| `list_unit_price` | numeric(18,4) | yes | Snapshot. Null only for a manual price |
| `floor_unit_price` | numeric(18,4) | yes | Snapshot |
| `quoted_unit_price` | numeric(18,4) | no | `CHECK >= 0`. "Your price" — defaults to list |
| `line_value` | numeric(18,4) | no | `GENERATED ALWAYS AS (round(quantity * quoted_unit_price, 4)) STORED` |
| `list_line_value` | numeric(18,4) | yes | Generated likewise from `list_unit_price` |
| `line_variance_pct` | numeric(7,4) | yes | Generated: `(quoted − list) / list × 100`, null when list is null or 0 |
| `taxable_value` / `cgst_amount` / `sgst_amount` / `igst_amount` | numeric(18,4) | no | Per line — GST is per HSN rate |
| `is_delisted` | boolean | no | `DEFAULT false`. Set by reprice when the new list has no price (MQT-10). Flagged, never deleted |

No `version` column: lines are replaced as a set by `PUT …/lines` under the parent's `version`.

#### `quotation_approval` — the decision record

One row per submission that needed approval. A version rejected and resubmitted has two rows.

| Column | Type | Null | Notes |
|---|---|---|---|
| `id` | uuid | no | pk |
| `quotation_version_id` | uuid | no | FK |
| `status` | text | no | `CHECK IN ('PENDING','APPROVED','REJECTED','WITHDRAWN')` |
| `reasons` | jsonb | no | The evaluation items that fired, frozen at submit — "every trigger that fired, not just the first" |
| `governing_variance_pct` | numeric(7,4) | yes | |
| `required_permission` | text | no | From the matched band. A permission, never a role (BATD P3) |
| `zone_id` | uuid | no | Copied from the chain, so the queue scopes without a join |
| `requested_by` / `requested_at` | uuid / timestamptz | no | |
| `decided_by` / `decided_at` | uuid / timestamptz | yes | |
| `decision_reason` | text | yes | Mandatory on reject; optional on approve |
| `created_at` | timestamptz | no | No `version`, no `updated_*` |

```sql
ALTER TABLE quotation_approval ADD CONSTRAINT qa_decision_recorded CHECK (
  (status = 'PENDING' AND decided_by IS NULL AND decided_at IS NULL)
  OR (status IN ('APPROVED','REJECTED','WITHDRAWN') AND decided_by IS NOT NULL AND decided_at IS NOT NULL)
);
ALTER TABLE quotation_approval ADD CONSTRAINT qa_reject_has_reason CHECK (
  status <> 'REJECTED' OR length(btrim(coalesce(decision_reason, ''))) > 0
);
CREATE UNIQUE INDEX quotation_approval_one_pending
  ON quotation_approval (quotation_version_id) WHERE status = 'PENDING';
CREATE INDEX quotation_approval_queue_idx
  ON quotation_approval (requested_at) WHERE status = 'PENDING';   -- oldest first
```

### 3.3 Things this does not add

- **No `approval_matrix_band` table.** §5.3.
- **No `contact_link` row for the attention contact.** `contact_link.entity_type`'s CHECK has no
  `QUOTATION`; the version holds `attention_contact_id` directly, which is all the screens need.
- **No quotation column on `opportunity`.** The opportunity list's quote chip (screen M2) is served
  by `GET /quotations/summary?opportunityIds=` from this module. Opportunities stays ignorant of
  quotes.

### 3.4 Master list values

None new. Payment and delivery terms are tables (0027). Rejection reasons are free text, as the
wireframe draws them (MQT-08); a `QUOTE_REJECTION_REASON` list is a later choice, not a gap.

---

## 4. Ports

### 4.1 Published by `pricing`

```ts
// src/modules/pricing/ports.ts — consumers re-declare these; they do not import them
export interface PriceQuery {
  readonly skuCode: string;
  readonly quantity: string;            // numeric as string, the house convention
  readonly asOf: string;                // IST business date
  readonly segment: 'B2B' | 'B2G';
  readonly customerCategoryId: string | null;
  readonly zoneId: string | null;
  readonly priceListId?: string;        // pin to a list (re-reading a draft); else the live one
}
export type PriceResolution =
  | { readonly kind: 'PRICED'; readonly priceListId: string; readonly priceListVersionNo: number;
      readonly priceListItemId: string; readonly unitPrice: string; readonly floorPrice: string | null;
      readonly bandMinQty: string; readonly priceBasis: string; readonly alwaysRequiresApproval: boolean }
  | { readonly kind: 'UNPRICED'; readonly priceListId: string | null; readonly reason:
      'NO_LIVE_LIST' | 'SKU_NOT_ON_LIST' };

export interface LiveListSummary {
  readonly priceListId: string; readonly versionNo: number; readonly effectiveFrom: string;
  readonly priceBasis: string; readonly alwaysRequiresApproval: boolean;
  readonly printedNotes: readonly string[];
}

export interface PriceReadPort {
  resolve(tx: Executor, q: PriceQuery): Promise<PriceResolution>;
  resolveMany(tx: Executor, qs: readonly PriceQuery[]): Promise<PriceResolution[]>;  // one round trip per draft
  liveList(tx: Executor, slice: { segment: string; customerCategoryId: string | null;
    zoneId: string | null; asOf: string }): Promise<LiveListSummary | null>;
  searchPriced(tx: Executor, q: { text: string; segment: string; customerCategoryId: string | null;
    zoneId: string | null; asOf: string; quantity: string | null; limit: number }):
    Promise<Array<{ skuCode: string; displayName: string; uom: string; sizeLabel: string | null;
      unitPrice: string }>>;   // MQT-03: only SKUs that carry a price
}
```

Port name `priceRead`. Resolution order when several lists are live: the most specific slice wins
(category and zone both set, then category, then zone, then neither). With one national B2B list
this reduces to one lookup.

### 4.2 New in `opportunities` (P-1)

```ts
export interface OpportunityForQuoting {
  readonly id: string; readonly refNo: string; readonly name: string;
  readonly organisationId: string; readonly siteId: string;
  readonly stateCode: string; readonly zoneId: string;
  readonly ownerUserId: string; readonly stageCode: string; readonly stageSortOrder: number;
  readonly outcome: 'WON' | 'LOST' | null;
}
export interface OpportunityReadPort {                 // extends the existing port
  existsForLead(tx: Tx, leadId: string): Promise<{ id: string; refNo: string } | null>;
  getForQuoting(tx: Tx, scope: Scope, id: string): Promise<OpportunityForQuoting | null>;
  stageSortOrder(tx: Tx, stageCode: string): Promise<number | null>;
}
export interface OpportunityStagePort {                // new, port name 'opportunityStage'
  advanceOrExplain(tx: Tx, input: { opportunityId: string; targetStageCode: string;
    actorId: string; reason: string }):
    Promise<{ advanced: true } | { advanced: false; blockedBy: FieldError[] }>;
}
```

`advanceOrExplain` is MBD-01 §2.10's `transitionOrExplain` under its port name. It never moves an
opportunity backwards: if the opportunity is already at or past the target, it answers
`{ advanced: true }` and writes nothing.

**Quoting opens at a stage** (OD-1, MO-02′ empty state). Quotations compares
`stageSortOrder(opportunity)` with `stageSortOrder(config quotation.openingStage)`. No stage code
appears in quotations' source.

### 4.3 Consumed by `quotations`

| Port | Publisher | Method | Used for |
|---|---|---|---|
| `opportunityRead` | opportunities | `getForQuoting`, `stageSortOrder` | Create, gate, refresh addressee |
| `opportunityStage` | opportunities | `advanceOrExplain` | Issue (§8.4), in the issuing transaction |
| `customerRead` | masters | `getSiteForQuoting` (P-2) | Name, GSTIN, state, category, address |
| `skuRead` | masters | `resolveForQuoting` (exists) | Description, UOM, HSN, rate, MOQ, `isQuotable`, phase-out replacement |
| `termsRead` | masters | `getPaymentTerm`, `getDeliveryTerm` (widened, P-3) | Term snapshot, approval flags, lead band |
| `contactRead` | contacts | `getSummary` (exists) | Attention name and designation |
| `priceRead` | pricing | all four | List price, staleness, reprice, picker |

### 4.4 Published by `quotations`

| Port | Consumer | Why |
|---|---|---|
| `attachmentOwner:QUOTE_PDF` | attachments | `GET /attachments/{id}/url` resolves the owner and applies **its** read scope. Without the port the download fails closed with a 404. Add `QUOTE_PDF` to `BOUND_OWNER_TYPES` in `attachment-owner-ports.test.ts` |

Nothing else. No other module needs to read a quote.

---

## 5. Domain components

Pure, no I/O, unit-tested in isolation; `domain-is-pure` enforces it. Each takes data in and
returns data out — every port call happens in the application layer.

| Component | Module | Responsibility |
|---|---|---|
| `QuoteCalculator` | quotations | Lines + tax result + freight → every total on the version. The only place money is summed |
| `VarianceCalculator` | quotations | Line and total variance, worst line, with the price-basis check |
| `ApprovalEvaluator` | quotations | Version + config → `Evaluation` (all items, pass or fail) and an outcome (§5.2) |
| `TaxPolicy` | quotations | Interface; `StubTaxPolicy` now, `GstTaxPolicy` after OD-4a (§5.4) |
| `QuoteStateMachine` | quotations | Legal transitions (§8). Refusals are 409s naming the current state |
| `ValidityPolicy` | quotations | Default and maximum validity, `isExpired` on read |
| `PriceListValidator` | pricing | Parsed sheet + aliases + SKU master + live list → validation report (§8.1) |
| `BandResolver` | pricing | Quantity + bands → band, with the edge rule from config |

### 5.1 Variance

```
line variance  = (quoted − list) / list × 100          null when list is null or 0
total variance = (Σ quoted·qty − Σ list·qty) / Σ list·qty × 100
worst line     = min(line variances)
governing      = worst line   if quotation.approval.evaluationLevel = 'LINE'   (seeded, BQ-22)
               = total        if 'TOTAL'
```

**Price basis.** If the list's `price_basis` and the delivery term's differ — an ex-works list on a
delivered quote — the variance is not comparable without grossing up freight, and the spec's §4.2
rule for that is not available (see the note at the top). Until it is, the evaluator does not guess:
it raises `PRICE_BASIS_MISMATCH`, which routes. `UNKNOWN` on the list always mismatches. This makes
the Sep sheet route every quote whose term is not `PRICE_EXCLUSIVE` until OD-3a is answered — which
is the honest consequence of not knowing.

### 5.2 Approval evaluation

The evaluator returns **every** check, passed or not — MQT-07 draws the passed ones with a tick —
and an outcome of `CLEARS`, `NEEDS_APPROVAL` or `BLOCKED`.

| Code | Fires when | Effect | Source |
|---|---|---|---|
| `BELOW_FLOOR` | any line's quoted price < its `floor_unit_price` | **BLOCKED** — nobody can approve | REQ-QT-03, MQT-07c |
| `HARD_FLOOR_PCT` | governing variance < `quotation.approval.hardFloorPct` | **BLOCKED** | REQ-QT-03 (seeded null) |
| `LINE_BELOW_LIST` | governing basis LINE and a line < list | routes | REQ-QT-03 |
| `TOTAL_BELOW_LIST` | governing basis TOTAL and total < list | routes | REQ-QT-03 |
| `PAYMENT_TERM_FLAGGED` | `payment_term.requiresApproval` | routes | spec §7.3 via `0027` |
| `LEAD_TIME_OVER_BAND` | `lead_time_days > delivery_term.maxLeadDays` | routes | spec §7.3 via `0027` |
| `ADDITIONAL_CONDITIONS` | `additional_conditions` non-empty, or an open-ended term | routes | MQT-06, OD-4c |
| `UNREGISTERED_BUYER` | `customer_gstin` is null | routes | spec §10.2 via `masters/ports.ts` |
| `LIST_POLICY` | the list's `always_requires_approval` | routes | Sep sheet note 3, Q-6 |
| `PRICE_BASIS_MISMATCH` | §5.1 | routes | OD-3a |
| `MANUAL_PRICE` | a line has no list price (if Q-3 allows it at all) | routes | price list Q-3 |

Each code can be switched off in `quotation.approval.triggers` without a release. The band matched
by the governing variance decides `required_permission`; when only non-price triggers fire, the
lowest band that requires approval supplies it.

### 5.3 Bands are a config key

```jsonc
// quotation.approval.bands — seeded, matches BATD §3.12's 06_approval_matrix
[
  { "min": 0,    "max": null, "outcome": "AUTO",    "permission": null },
  { "min": null, "max": 0,    "outcome": "APPROVE", "permission": "quotation:approve" }
]
```

`[min, max)`, evaluated in order, first match wins, null = unbounded. Zod-validated: ordered,
non-overlapping, permissions must exist in the catalogue. BATD defined the matrix twice — as this
key and as `approval_matrix_band`. The config registry already gives history, audit, an editor
screen and `pg_notify` reload; a table would duplicate all four. Salespeople do not hold
`config:read`, so the bands and floor stay hidden from them, as `role-definitions.ts` intends.

### 5.4 Tax

```ts
interface TaxPolicy {
  readonly key: string;                       // written to quotation_version.tax_policy
  compute(input: { supplierStateCode: string; placeOfSupplyStateCode: string;
    lines: ReadonlyArray<{ taxableValue: string; gstRatePct: string }>;
    freight: { amount: string; gstRatePct: string | null } }): TaxResult;
}
```

- **`GstTaxPolicy`** — intra-state when `supplierStateCode = placeOfSupplyStateCode` (CGST + SGST,
  half each), else IGST at the full rate. Per line, rounded to the paisa half-up, totals summed from
  rounded lines. **Both state codes are `IN-xx`** — the string compare that would have made every
  quote IGST cannot happen, because both come from `state(code)`.
- **`StubTaxPolicy`** (`STUB_V0`) — the same arithmetic, with `supplierStateCode` from config and a
  banner on every screen and PDF that tax is provisional. Selected by `quotation.tax.policy`.
- OD-4a still has to answer: Atomberg's GSTIN per supplying state (one supplier state or several),
  freight's tax treatment, and rounding (M2 §22.3). Each is a policy change, not a schema change.

### 5.5 Price list band edges

`pricing.import.bandEdges` — seeded `[1, 201, 501]` with `pendingDecision: 'price list Q-1'`.
`quotation.pricing.bandQuantityBasis` — `LINE` or `ORDER`, seeded `LINE` with `pendingDecision:
'price list Q-2'`. Under `ORDER`, quotations sums quantities per price-list row before resolving;
the port does not change.

---

## 6. API contract

All mutations take `Idempotency-Key` (global middleware) and draft writes take the version in
`If-Match`. All reads keyset-paged. Request schemas defined once in `api/schemas.ts` and shared by
route and contract (P75's 27 drifts).

### 6.1 `quotations` — 20 operations

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/opportunities/:id/quotations` | `quotation:read` | Chains with their versions, newest first (MQT-01) |
| POST | `/opportunities/:id/quotations` | `quotation:create` | New chain + v1 DRAFT. 409 `QUOTING_NOT_OPEN` naming the opening stage |
| GET | `/quotations/summary` | `quotation:read` | `?opportunityIds=` — status chip for the opportunity list (M2) |
| GET | `/quotations/:id` | `quotation:read` | Chain + version list |
| GET | `/quotations/:id/versions/:no` | `quotation:read` | Full version, lines, evaluation, `isExpired`, `isStale` |
| PATCH | `/quotations/:id/draft` | `quotation:update` | Header: terms, lead time, conditions, validity, attention, freight. Returns recomputed totals |
| PUT | `/quotations/:id/draft/lines` | `quotation:update` | Replace the line set. Debounced by the client; one `resolveMany` |
| GET | `/quotations/:id/draft/evaluation` | `quotation:read` | Preview of §5.2 — MQT-07 before submit |
| GET | `/quotations/:id/draft/reprice` | `quotation:read` | Line-by-line old list → new list (MQT-10) |
| POST | `/quotations/:id/draft/reprice` | `quotation:update` | Moves `list_unit_price`, **never** `quoted_unit_price`; flags delisted lines |
| POST | `/quotations/:id/draft/submit` | `quotation:update` | Evaluate → issue now, or raise approval (§8.3). 409 `PRICE_LIST_MOVED` if stale |
| POST | `/quotations/:id/draft/withdraw` | `quotation:update` | PENDING_APPROVAL → DRAFT; approval row → WITHDRAWN |
| POST | `/quotations/:id/draft/cancel` | `quotation:update` | DRAFT → CANCELLED |
| POST | `/quotations/:id/revise` | `quotation:create` | New DRAFT version priced on today's list |
| GET | `/quotations/:id/versions/:no/pdf` | `quotation:read` | `{ attachmentId, status }`; the URL comes from `/attachments/{id}/url` |
| GET | `/quotations/sku-search` | `quotation:read` | `?q=&quotationId=&quantity=` — priced SKUs only (MQT-03) |
| GET | `/quotation-approvals` | `quotation:approve` | `?status=PENDING`, oldest first, reasons as chips (S9) |
| GET | `/quotation-approvals/:id` | `quotation:approve` | Lines with quoted, list, variance; terms; conditions shown apart (S10) |
| POST | `/quotation-approvals/:id/approve` | from the row's `required_permission` | Optional reason. Issues in the same transaction |
| POST | `/quotation-approvals/:id/reject` | likewise | Reason mandatory — 422 without one (T-M2-19) |

**404, not 403, for a quote outside scope** — the house rule.

### 6.2 `pricing` — 11 operations

| Method | Path | Permission | Notes |
|---|---|---|---|
| GET | `/price-lists` | `pricelist:read` | |
| GET | `/price-lists/live` | `pricelist:read` | `?segment=` |
| GET | `/price-lists/:id` | `pricelist:read` | Header + validation report |
| GET | `/price-lists/:id/items` | `pricelist:read` | `?sku=&q=` |
| POST | `/price-lists` | `pricelist:manage` | `{ attachmentId, segment, effectiveFrom, … }` → DRAFT, parsed |
| POST | `/price-lists/:id/validate` | `pricelist:manage` | → VALIDATED or the blocking report |
| POST | `/price-lists/:id/publish` | `pricelist:manage` | → SCHEDULED (future date) or LIVE; supersedes the previous |
| POST | `/price-lists/:id/withdraw` | `pricelist:manage` | DRAFT or VALIDATED only |
| GET | `/price-source-aliases` | `pricelist:read` | `?unmatched=true` lists labels from the last upload with no alias |
| POST · PATCH | `/price-source-aliases[/:id]` | `pricelist:manage` | |
| POST | `/price-source-aliases/:id/deactivate` | `pricelist:manage` | **Not `DELETE`** |

Add both modules to `MODULES` in `registry.ts`, to `ALL_ROUTE_DOCS` in `tools/route-manifest.ts`, and
to `test/guards/openapi-complete.test.ts` **in the same commit as the first route** — the guard
silently skips a module it does not import.

---

## 7. Authorization and visibility

### 7.1 Permissions

Flat `{resource}:{action}`.

| Permission | Grants | Seeded to |
|---|---|---|
| `quotation:read` | Read quotes in scope | SALESPERSON, ADMIN, VIEWER |
| `quotation:create` | New chain, revise | SALESPERSON |
| `quotation:update` | Edit own draft, submit, withdraw, cancel, reprice | SALESPERSON |
| `quotation:approve` | The queue and its decisions | ADMIN |
| `quotation:override_tax` | Change place of supply (MQT-02, OD-4a), audited | ADMIN |
| `pricelist:read` | Read lists and items | SALESPERSON, ADMIN |
| `pricelist:manage` | Upload, validate, publish, withdraw, aliases | ADMIN |

BATD's `quotation:submit` and `quotation:issue` are folded: submitting is editing your draft, and
issue is never a separate human act — it happens on a clean submit or an approval. **Self-approval**
(Q-M2-20): refused in the use case — `decided_by ≠ requested_by` — whatever permissions the
approver holds. A rule, not a permission, because a permission can be granted around.

### 7.2 Scope

- **Read:** `readScope(ctx)` — the caller's zones, or all with `zone:all`. Columns
  `{ zone: 'q.zone_id', owner: 'q.owner_user_id' }` on `quotation`. Versions, lines and approvals
  are reached only through a scoped `quotation` read; no repository reads them directly.
- **Write:** `writeScope(ctx)` — owned. A salesperson edits only their own drafts.
- **Approval queue:** `readScope` on `quotation_approval.zone_id`.
- `every-repo-scoped` applies to both modules. Masters-style national data (`price_list`) is not
  zone-scoped (D-22).

---

## 8. Flows

### 8.1 Price list: upload → validate → publish

```
Admin                                  pricing
  | POST /attachments … complete   →   file stored (platform route)
  | POST /price-lists              →   DRAFT; parse rows with platform/csv importCsv (xlsx → rows)
  | POST /price-lists/:id/validate →   PriceListValidator:
  |                                     - every label has an approved alias        (else: unmatched list)
  |                                     - every alias resolves to ≥1 quotable SKU
  |                                     - prices numeric, ≥ 0, bands non-increasing (warn, not block)
  |                                     - diff vs live: added, removed, changed > pricing.changeThresholdPct
  |                                     - SKUs on the live list missing from this one
  |                                   blocking issues → stays DRAFT, report returned (T-M2-11)
  |                                   none           → VALIDATED, items exploded per SKU
  | POST /price-lists/:id/publish  →   effective today  → LIVE, previous LIVE → SUPERSEDED (effective_to set)
  |                                    effective later  → SCHEDULED; job activates it (§10)
  |                                    emit PriceListPublished
```

The report is **blocking**, not advisory (M2 F-M2-13). Publishing never touches a quote: drafts
discover the new list through staleness on their next read.

### 8.2 Build a draft

```
POST /opportunities/:id/quotations
  opportunityRead.getForQuoting(scope)       404 outside scope
  gate: stage ≥ quotation.openingStage       409 QUOTING_NOT_OPEN { openingStage }
  gate: outcome is null                      409 OPPORTUNITY_CLOSED
  customerRead.getSiteForQuoting(siteId)     → segment, GSTIN, state, category, address
  priceRead.liveList(slice)                  409 NO_LIVE_PRICE_LIST if none
  termsRead: segment defaults                → payment and delivery term snapshots, lead time
  insert quotation + v1 DRAFT

PUT /quotations/:id/draft/lines  (If-Match: version)
  skuRead.resolveForQuoting per SKU          422 per line: not quotable, no GST rate, discontinued
                                             (PHASE_OUT warns with the replacement)
  priceRead.resolveMany(pinned to price_list_id)
                                             UNPRICED → 422 PRICE_UNRESOLVED naming the SKU (Q-3)
  quoted_unit_price defaults to list
  TaxPolicy → QuoteCalculator → VarianceCalculator → ApprovalEvaluator (preview)
  replace lines, update totals + evaluation, version + 1
  ← totals, per-line variance, evaluation    (MQT-04 shows "6.1% below list. This will need approval.")
```

The app computes nothing. Every figure on every screen is from this response.

### 8.3 Submit

```
POST /quotations/:id/draft/submit
  stale?  priceRead.liveList ≠ price_list_id     → 409 PRICE_LIST_MOVED (MQT-10 offers reprice)
  re-read terms and GST rates; recompute; ApprovalEvaluator
  BLOCKED         → 422 with the evaluation; nothing written           (MQT-07c)
  CLEARS          → §8.4 issue, in this transaction                    (MQT-07a)
  NEEDS_APPROVAL  → version PENDING_APPROVAL; quotation_approval PENDING with reasons;
                    emit QuoteSubmittedForApproval                      (MQT-07b, MQT-08)
```

### 8.4 Issue

The same code path from a clean submit and from an approval:

```
withTransaction:
  quotation.quote_no ← allocate(tx, series)  only if null — revisions keep the chain's number
  version: status ISSUED, issued_at/by, pdf_status PENDING, terms snapshot frozen
  previous ISSUED version on the chain → SUPERSEDED                   (predecessor stays live until now)
  opportunityStage.advanceOrExplain(target = config quotation.issue.advanceToStage)
     blocked → the issue still stands; response carries blockedBy     (A-M2-10, C-M2-04)
  recordAuditEvent ISSUE
  emit QuoteIssued                                                    (→ render job, → notification)
after commit:
  quotation.pdf.render job: platform renderer → AttachmentService.putDirect(QUOTE_PDF, sensitive=false)
     → pdf_attachment_id, pdf_checksum, pdf_status READY   (FAILED is retried; never re-rendered once READY)
```

The PDF is rendered once from the frozen version and never regenerated (REQ-QT-04). An issued
version two years later shows the same names, prices, tax rate and price list version, because
none of them is looked up.

### 8.5 Approve, reject, withdraw, revise

| From | Action | To | Notes |
|---|---|---|---|
| PENDING_APPROVAL | approve | ISSUED | §8.4 in the same transaction. Approval row APPROVED. `QuoteApproved` |
| PENDING_APPROVAL | reject | **DRAFT, same version** | Approval row REJECTED with reason. `QuoteRejected`. MQT-08: "Editing keeps the same quotation" |
| PENDING_APPROVAL | withdraw | DRAFT | Approval row WITHDRAWN (A-M2-09) |
| DRAFT | submit | PENDING_APPROVAL or ISSUED | §8.3 |
| DRAFT | cancel | CANCELLED | |
| ISSUED (latest) | revise | new version DRAFT | Priced on today's list (A-M2-11); approved independently (A-M2-12). `QuoteRevised` |
| ISSUED | successor issued | SUPERSEDED | Never directly |

### 8.6 Immutability

Two triggers in `0042b`, in the `attendance_day` pattern, because grants cannot do it (one role
owns every table since P30):

- **`quotation_version`:** once not DRAFT, only these changes are allowed — `status` along the
  transitions above, `pdf_attachment_id`/`pdf_checksum` null → set, `pdf_status` PENDING/FAILED →
  READY/FAILED. Anything else raises. **`quotation_line`:** no insert, update or delete when the
  parent version is not DRAFT.
- **`quotation_approval`:** once decided, nothing changes. Never deleted.
- **`price_list_item`:** nothing changes once the parent list is not DRAFT.

---

## 9. Configuration keys

Declared in each module's `config.ts`. `pendingDecision` on every provisional value.

| Key | Default | Pending |
|---|---|---|
| `quotation.enabled` | `false` | Go-live flag. Build and merge freely |
| `quotation.openingStage` | the ENQUIRY stage code (seeded value, not source) | OD-1 |
| `quotation.issue.advanceToStage` | the QUOTE_SUBMITTED stage code | — |
| `quotation.validity.defaultDays` | 30 | REQ-QT-02 |
| `quotation.validity.maxDays` | 90 | — |
| `quotation.approval.bands` | §5.3 | OD-4b |
| `quotation.approval.evaluationLevel` | `LINE` | OD-4b, BQ-22 |
| `quotation.approval.hardFloorPct` | `null` | REQ-QT-03 |
| `quotation.approval.triggers` | all on except `LIST_POLICY` | OD-4c, Q-6 |
| `quotation.approval.ageing.warnAfterHours` | 24 | RSK-01 |
| `quotation.tax.policy` | `STUB_V0` | OD-4a |
| `quotation.tax.supplierStateCode` | `null` — stub refuses to compute until set | OD-4a |
| `quotation.numbering.seriesBySegment` | `{ "B2B": "QUOTE", "B2G": "QUOTE" }` | Q-7 |
| `quotation.pricing.bandQuantityBasis` | `LINE` | price list Q-2 |
| `quotation.pdf.printListNotes` | `false` | — |
| `pricing.import.bandEdges` | `[1, 201, 501]` | price list Q-1 |
| `pricing.changeThresholdPct` | 10 | Q-M2-17 |

**Number format.** The seed has `QT/{fy}/{seq:05}`; the wireframes draw `ATB/B2G/26-27/00041`. Both
are data. If the business picks the wireframe, the answer is two `number_series` rows
(`QUOTE_B2G` prefix `ATB/B2G`, `QUOTE_B2B` prefix `ATB/B2B`) and a config write — no code.

---

## 10. Jobs and events

| Job | Module | Trigger | Does |
|---|---|---|---|
| `quotation.pdf.render` | quotations | On `QuoteIssued` | §8.4. Retries with backoff; FAILED is visible on the version |
| `quotation.approval.ageing` | quotations | Hourly | Emits `ApprovalRequestAged` for PENDING rows past the threshold, once per threshold crossing |
| `quotation.validity.reminder` | quotations | Daily | Notifies the owner N days before `validity_date` (MH-01′ "expires in 3 days") |
| `pricing.list.activate` | pricing | Daily, 00:05 IST | SCHEDULED → LIVE when due; previous → SUPERSEDED; `PriceListPublished` |

Expiry is not a job (§3.2). **Worker state is the same as MBD-06 §10:** jobs are not scheduled on
the deployed environment yet. Nothing in a quote's correctness depends on a job — a missed reminder
is a missed reminder, and a missed activation leaves the old list live, which the price list screen
shows.

**Events** (already declared): `PriceListPublished`, `QuoteSubmittedForApproval`, `QuoteApproved`,
`QuoteRejected`, `QuoteIssued`, `QuoteRevised`, `ApprovalRequestAged`. Payloads carry ids, never
amounts — a subscriber that needs figures reads them.

| Event | Notifies |
|---|---|
| `QuoteSubmittedForApproval` | Holders of the row's `required_permission` in its zone |
| `ApprovalRequestAged` | The same, and the submitter |
| `QuoteApproved` / `QuoteRejected` | The owner — the Today screen's "Needs you" |
| `PriceListPublished` | Admins; salespeople with an open draft on the superseded list |

In-app first (P-5); push when the device-token path is wired.

---

## 11. Migrations

Continuing from `0039`. Forward-only, unqualified, `SET search_path FROM CURRENT` on functions,
`attach_audit` at the end of each creating file, no `INSERT … VALUES` (AD-16 — seeds are TypeScript).

| File | Contents |
|---|---|
| `0040_price_list.sql` | `price_list`, `price_list_item`, `price_source_alias`, the exclusion constraint, indexes |
| `0040b_price_list_immutable.sql` | Item immutability once the list leaves DRAFT |
| `0041_quotation.sql` | `quotation`, `quotation_version`, `quotation_line`, indexes, `attach_zone_rls` on `quotation` |
| `0042_quotation_approval.sql` | `quotation_approval` |
| `0042b_quotation_immutable.sql` | §8.6 triggers |
| `db/seeds/15_pricing_aliases.ts` | Loads `db/data/price-source-aliases.csv` — 101 labels, 89 heuristic + 12 manual, all `approved_by` SYSTEM pending review |
| `db/seeds/16_quotation_provisional.ts` | Number series rows if Q-7 lands; nothing otherwise |

`npm run codegen` after `0042b`, or `generated-types-current` fails. Add `'QUOTE_PDF'` to the
`attachment.owner_type` comment list. `no-role-literals` scans `.sql` — no role names in triggers.

**The Sep 2026 list is not a seed.** It is uploaded through the API like every later list, so the
first real upload exercises the real path. Demo data (§13, phase 10) uploads it.

---

## 12. Guards

| Guard | Asserts |
|---|---|
| `quotations-never-read-prices` | No file under `modules/quotations/` references `price_list_item` or `price_list` in SQL |
| `quote-money-single-sum` | Totals are computed only in `QuoteCalculator` |
| `no-quote-mutation-after-issue` | The trigger rejects an update to an ISSUED version's lines, terms or totals |
| `quote-number-at-issue` | `allocate(…, 'QUOTE…')` is called only from the issue use case |
| `approval-reasons-complete` | Every evaluation item code has a label and a config switch |

Existing guards that apply unchanged: `no-cross-module-import`, `no-stage-literals`,
`no-role-literals`, `openapi-complete`, `audit-coverage`, `every-mutation-idempotent`,
`validation-returns-all`, `generated-types-current`, `every-repo-scoped`, `domain-is-pure`,
`attachment-owner-ports`.

---

## 13. Build sequence

Each phase stops for a commit. Phases are ordered so that **nothing waits on a business answer
before phase 9**.

| Phase | Contents | Gated on |
|---|---|---|
| **0** | This document reviewed; the external specification checked against §5.2 and §8 | — |
| **1** | P-1 `opportunities` ports · P-2, P-3 `masters` ports — three small branches | — |
| **2** | `0040`/`0040b`, `pricing` module: upload, validate, publish, `priceRead`, aliases seed, `codegen` | Phase 1 |
| **3** | `0041`–`0042b`, `codegen` | Phase 2 |
| **4** | Domain components (§5) with `StubTaxPolicy`; config keys | Phase 3 |
| **5** | `quotations` skeleton: create, draft header, lines, evaluation preview, SKU search, summary | Phase 4 |
| **6** | Submit, approve, reject, withdraw, issue, numbering, stage advance, events | Phase 5 |
| **7** | P-4 renderer, `quotation.pdf.render`, `attachmentOwner:QUOTE_PDF` — placeholder template | Phase 6 |
| **8** | Revise, reprice, staleness; P-5 notifications; the three jobs | Phase 6 |
| **9** | `GstTaxPolicy`; band edges and basis set; floor and triggers set | **OD-4a, price list Q-1/Q-2, OD-4b/4c** |
| **10** | Demo data (upload the Sep list, ten quotes across every state), one honest test figure | Phase 8 |
| — | **Mobile** — `hermes-app-client` `feat/quotation` from mocks to the API; remove the flag | Phase 6 for W3, phase 8 for W4 |
| — | **Portal** — price list desk, approvals queue | **Wireframes not drawn** |

The mobile app's quotation screens already exist on mocks (MQT wireframes, S1–S10). The contract in
§6 should be co-signed by the app lead before phase 5, as the app screen spec requires.

---

## 14. Test plan

- **Unit:** `QuoteCalculator` reconciling to a hand calculation intra- and inter-state;
  `VarianceCalculator` at exactly list, ₹1 below, and a zero list price; `ApprovalEvaluator` for
  every row of §5.2 alone and combined; `BandResolver` at 1, 200, 201, 500, 501;
  `PriceListValidator` against the real Sep sheet (101 rows, 12 manual aliases, flat rows, the
  trailing-space labels).
- **Integration:** the full cycle — upload, validate, publish, draft, submit clean, issue, PDF;
  submit discounted, reject, edit, resubmit, approve; revise and the predecessor superseding only on
  the successor's issue; a list published mid-draft blocking submit until reprice; the triggers
  rejecting a write to an issued version; self-approval refused.
- **Contract:** all 31 operations (20 + 11).
- **Scope:** another salesperson's quote is a 404; an Admin in SOUTH does not see NORTH's queue
  without `zone:all`.
- **Acceptance (M2 §21):** T-M2-11 to T-M2-27 map one-to-one onto the above; each named in its test.

Target **~110 new tests**. Isolation by run suffix works here — no column is uniquely indexed on a
natural key the suffix cannot vary.

---

## 15. Provisional and open

### Settled by this document

| Item | Position |
|---|---|
| Module split | `pricing` + `quotations` (§1.3) |
| Price list shape | Dimensions on the header, bands as rows, exploded per SKU via aliases (§3.1) |
| Place-of-supply spelling | FK to `state(code)` (§3.2) |
| Bands: table or config | Config (§5.3) |
| Expiry | Derived (§3.2) |
| Rejection | Same version back to DRAFT (§8.5) |
| Several quotes per opportunity | Allowed by the schema; one index if Q-M2-05 says otherwise |

### Still open — business

| Item | Owner | Blocks |
|---|---|---|
| Price list Q-1: which band holds quantity 200 | Sales Excellence | Phase 9 (config) |
| Q-2: band on line or order quantity | Sales Excellence | Phase 9 (config) |
| Q-3: unpriced SKU — blocked, or manual price that routes | Sales Excellence | `MANUAL_PRICE` on or off |
| Q-4: five rows flat across bands — deliberate | Sales Excellence | Data only |
| Q-5: B2B only, B2G priced separately | Sales Excellence | A second list, not a migration |
| Q-6: note 3, approval on every B2B order | Sales Excellence | `LIST_POLICY` on or off |
| Note 1: 100% advance on every B2B order vs the wireframe's 30/70 default | Sales Excellence | Default payment term per segment |
| OD-3a: list ex-works or delivered | Sales Excellence | `price_basis`; `PRICE_BASIS_MISMATCH` |
| OD-4a: GST — supplier state(s), freight tax, rounding | Sales Excellence + Finance | Phase 9 |
| OD-4b: line or total | Sales Excellence | Config |
| OD-4c: the full trigger list | Unassigned — not in the PRD register | Config |
| OD-1: stage at which quoting opens | B2G/B2B Sales | Config |
| Q-7: number format | Sales | Config + two series rows |
| BQ-47: customer category list | Sales Excellence | Category-specific lists only |
| Q-M2-05, -12, -20, -21, -23 | per M2 §19 | Index; reminder copy; nobody-acts escalation; cancelling an issued quote |
| PDF template (CT-06) | Sales + Marketing | Phase 7 ships a placeholder |
| Approver on mobile (wireframe Q-1, CR-01) | Product | Nothing in the backend — the API serves both |

### Still open — engineering

| Item | State |
|---|---|
| The external quotation specification | Not in the repo. §5.2 and §8 reconstruct it from code comments |
| MQT-11 term requests | A masters request flow; not this module |
| Worker scheduling on the deployed environment | As MBD-06 §10 |
| `QUOTE_SUBMITTED` described as "seeded inactive" in MBD-01 §0.2 | Active since P77; correct on reissue |

---

## 16. Cross-references to correct on reissue

- **BATD §3.7** — the `sales.` qualified DDL, per-item price dimensions, `specificity`,
  `approval_matrix_band` and `quote_status` enum are superseded by §3 here.
- **BATD §2.5.2** — `quotation:submit` and `quotation:issue` folded (§7.1); `quotation:read` and
  `quotation:override_tax` added.
- **BATD §3.11** — AUTO_APPROVE leading to an `APPROVED` status; here a clean submit issues.
- **M2 §6.2** — Rejected → new version; here, same version (the wireframe's later decision).
- **M2 §4.3** — `Expired` is derived, not stored.
- **MBD-01 §2.10** — `transitionOrExplain` is published as `opportunityStage.advanceOrExplain`.
- **`number_series` DDL comment** — says `{seq:06}`; the seed and code use `{seq:05}`.
- **decisions-log** — add OD-4c to the register; it currently lives only in build status.
