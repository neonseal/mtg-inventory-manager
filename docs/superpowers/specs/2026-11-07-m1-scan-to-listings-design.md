# MTG Inventory Manager — M1 "Scan → Listings" Design

- **Status:** Draft for review
- **Date:** 2026-11-07
- **Scope:** Overall architecture (all milestones) + detailed design for Milestone 1
- **Owner:** Corey Harris

---

## 1. Context

A single seller lists Magic: The Gathering singles on two marketplaces:

- **Manapool** — cards worth **$3.00 and up**
- **TCGplayer** — cards worth **under $3.00**

Cards are scanned with the **ManaBox** app, which exports a CSV. Converting that CSV into listings on both sites is today's largest time sink. This project builds a local web tool that turns a ManaBox scan into priced, reviewed listings on both sites, and that grows (in later milestones) into reconciliation, repricing, cross-site moves, and sales tracking.

### 1.1 Goals (whole project)

1. Turn a ManaBox scan into listings on the correct site with correct prices, with a human review step.
2. Keep a durable record of what was listed where, when, and at what price.
3. Reprice automatically using explicit, testable rules.
4. Move cards between sites as their value crosses the $3 line.
5. Track sales and profit.
6. Serve as a learning project for functional programming (Elixir/OTP) built to production standards.

### 1.2 Non-goals

- Multi-user / SaaS. One user, one seller account per site.
- Scraping any website. All data comes from official APIs, official exports, or open data providers.
- eBay, Cardmarket, CardTrader in M1–M5 (the architecture must not block CardTrader later).
- A native mobile app.

---

## 2. External constraints (researched 2026-11-05 → 11-07)

| System | Constraint | Consequence |
|---|---|---|
| **TCGplayer API** | Closed to new developers; existing keys only. | No API integration. TCGplayer is a **file-transport** marketplace. |
| **TCGplayer Seller Portal** | Level 4 sellers (user is Level 4) can *Export from Live*, *Export Filtered CSV* (any catalog products, incl. ones not stocked), and *Import to Staged*. Import changes quantity via **Add to Quantity** (signed delta); **Total Quantity** is reference-only; **TCG Marketplace Price** is always required. | Uploads are deltas, not totals. The tool generates staged-import CSVs and ingests exports. |
| **Manapool** | Open API with a per-account token (OpenAPI 3.1 spec published). Also imports ManaBox-format and TCGplayer-format CSVs. Docs mention a `custom_external_id` field. | Manapool is an **API-transport** marketplace; CSV is the fallback. Exact endpoints, identifiers and rate limits to be confirmed (spike T-0). |
| **ManaBox export** | Columns include Name, Set code, Collector number, Foil (`normal`/`foil`/`etched`), Rarity, Quantity, ManaBox ID, **Scryfall ID**, Purchase price (= ManaBox market price at scan time, USD), Condition (e.g. `near_mint`), Language (e.g. `en`), Misprint, Altered, Signed, Proxy. | Matching keys off **Scryfall ID**, which is present on every row. |
| **MTGJSON** | Open data. Card identifiers include Scryfall ID ↔ MTGJSON UUID. `TcgplayerSkus` maps UUID → list of {condition, language, printing, finishes, productId, skuId}. Known data issues exist for some alt-art non-English printings. `AllPrices` provides daily TCGplayer/Card Kingdom/Cardmarket prices with limited history. | Canonical identity and SKU resolution come from MTGJSON. A review queue is required for ambiguous matches. |
| **TCGTracking Open TCG API** | Free, no key. Per-set endpoints. SKU rows keyed by TCGplayer SKU ID with condition, variant, language, `mkt` (market), `low` (lowest listing), `hi`, `cnt` (active listing count), and `mp` (Manapool low, Magic only, when linked). Product-level Manapool lows for normal/foil/etched. Prices refresh daily (~9:35 AM ET); responses support ETag/Last-Modified. No published terms of use, rate limits, or SLA. | Primary automated price source, behind a fallback chain. The user will email TCGTracking to confirm personal use. |

---

## 3. Business rules

All numeric values are stored in `settings` and editable. Defaults shown.

### 3.1 Site assignment (on import)

- ManaBox price (`Purchase price`) **≥ $3.00 → Manapool**, otherwise **TCGplayer**.
- Exactly $3.00 goes to Manapool.
- Rows with no ManaBox price → `needs_review` (`no_scan_price`).
- The **$2.50 / $3.50 hysteresis band** applies only to moving cards that are already listed (M4), never to new imports.

### 3.2 TCGplayer price

```
candidates = [ low × 0.99, market × 0.85, floor ]
price      = round_down_to_cent(max(candidates))
```

- `low` = TCG lowest listing for the SKU; `market` = TCG Market Price for the SKU; `floor` = **$0.15**.
- Rounded **down** to the cent so the undercut stays under the low price. The result is never below `floor`.
- If `low` is missing, its candidate is dropped. If `market` is also missing → `needs_review` (`no_price`).
- If the low listing comes from fewer than **3** active listings (`cnt < 3`), the item is priced by the formula but flagged `thin_market`.
- No cosmetic rounding (e.g. to $x.99).

### 3.3 Manapool price (step-down)

| Days since listed | Multiplier on TCG Market |
|---|---|
| 0–14 | **1.05** |
| 15–30 | **1.02** |
| 31+ | **0.99** |

- `price = round_half_up_to_cent(market × multiplier)`.
- M1 only applies the day-0 multiplier (1.05). Weekly step-down repricing is M3.
- **Manapool-low guard:** if `price > manapool_low × 1.15`, flag `above_manapool_low` for review. Never changed automatically.
- **High-value guard:** any item with `price ≥ $50.00` (either site) is flagged `high_value` for manual review.

### 3.4 Price source chain (per price point)

1. TCGplayer *Export Filtered CSV* snapshot, if uploaded and less than **3 days** old.
2. TCGTracking daily snapshot.
3. MTGJSON `AllPrices` (TCGplayer retail) — market only.

If the newest usable snapshot for an item is older than 3 days, the item is flagged `stale_price` and the UI shows a banner.

---

## 4. Milestones

| # | Milestone | Depends on |
|---|---|---|
| **M1** | **Scan → listings** (this spec) | — |
| M2 | Reconciliation: sync both sites, detect sales and manual edits | M1 |
| M3 | Repricing: weekly Manapool step-down; TCGplayer reprice from fresh snapshots | M1, M2 |
| M4 | Cross-site moves using the $2.50 / $3.50 band | M2, M3 |
| M5 | Sales and profit tracking with charts | M2 |

Each later milestone gets its own spec → plan → tickets.

---

## 5. Technology

| Concern | Choice |
|---|---|
| Language / runtime | Elixir on the BEAM |
| Web / UI | Phoenix 1.8 + LiveView 1.2 |
| Background jobs / cron | Oban (Postgres engine) |
| Database | PostgreSQL in Docker (dev, test, and future prod) |
| Data access | Ecto (changesets validate CSV rows) |
| HTTP | Req |
| Manapool client | Generated from its OpenAPI spec with `oapi_generator`, executed through a Req-based client module |
| CSV | NimbleCSV (streaming) |
| Money | Integer cents + ISO currency code; `Decimal` for intermediate math; no floats |
| Tests | ExUnit, StreamData (property tests), `Req.Test` stubs, `Phoenix.LiveViewTest` |
| Quality | `mix format`, Credo (strict), Dialyzer, GitHub Actions CI |

Runs locally (`mix phx.server` + `docker compose up db`). Designed to be hostable later without restructuring.

---

## 6. Architecture

### 6.1 Principles

1. **Functional core, imperative shell.** Matching, pricing, site assignment and plan building are pure functions over plain data. Only adapters and job workers perform I/O.
2. **Plan, then apply.** Every outbound change (new listings now; reprices and moves later) is first computed as a **plan** of intended changes, reviewed, then applied. Applied plans are kept as reports.
3. **Sites are the source of truth for quantity; the tool keeps an append-only record** of what it listed, with timestamps and prices. Reconciliation (M2) compares the two.
4. **Idempotency everywhere.** External events carry unique keys; outbound pushes carry external references so retries never duplicate.

### 6.2 Contexts (Phoenix domain modules)

| Context | Responsibility | Purity |
|---|---|---|
| `Catalog` | Printings, variants, external IDs; MTGJSON ingest | I/O at ingest; lookups pure over loaded data |
| `Prices` | Price snapshots; `PriceSource` adapters; source-chain resolution | Adapters I/O; resolution pure |
| `Imports` | ManaBox parsing, row validation, batch lifecycle | Parsing pure; persistence I/O |
| `Matching` | Scryfall ID → printing → variant → SKU; `exact` / `ambiguous` / `unmatched` | Pure |
| `Pricing` | Site assignment, TCGplayer rule, Manapool rule, guards/flags | Pure |
| `Plans` | Build plans from (rows, matches, prices, settings); apply via marketplace adapters | Build pure; apply I/O |
| `Listings` | Append-only `listing_events`; derived current state | Writes I/O; derivation pure |
| `Marketplaces` | `Marketplace` behaviour; `Manapool` (API) and `TCGplayer` (file) adapters | I/O |
| `Settings` | Rule parameters | — |

### 6.3 Behaviours

```elixir
defmodule Marketplaces.Marketplace do
  @type transport :: :api | :file
  @callback transport() :: transport()
  @callback marketplace_id() :: atom()
  # Apply a list of plan items. :api adapters push and return per-item results;
  # :file adapters return a generated file and mark items :awaiting_confirmation.
  @callback apply_items([PlanItem.t()]) ::
              {:ok, [ItemResult.t()]} | {:ok, {:file, Path.t(), [ItemResult.t()]}} | {:error, term()}
end

defmodule Prices.PriceSource do
  @callback source_id() :: atom()
  # Fetch snapshots for the given sets; returns plain snapshot maps for persistence.
  @callback fetch(set_codes :: [String.t()], opts :: keyword()) ::
              {:ok, [Snapshot.t()]} | {:error, term()}
end
```

Adapters in M1: `Marketplaces.Manapool` (`:api`), `Marketplaces.TCGplayer` (`:file`); `Prices.TCGplayerExport`, `Prices.TCGTracking`, `Prices.MTGJSON`.

### 6.4 Identity model

- **Printing:** Scryfall ID + finish (`nonfoil` / `foil` / `etched`).
- **Variant:** printing + condition + language (our SKU-equivalent).
- **ExternalId:** variant → `{marketplace, external_id}` (e.g. `{:tcgplayer_sku, "1260372"}`).

Etched foils resolve to their own TCGplayer product (via MTGJSON `finishes`).

### 6.5 Supervision tree

```
MtgInventory.Application
├── MtgInventory.Repo
├── Phoenix.PubSub              # job progress → LiveView
├── Oban
│   ├── queue :catalog   (1)    # MTGJSON ingest
│   ├── queue :prices    (1)    # TCGTracking / MTGJSON prices
│   ├── queue :imports   (2)    # parse → match → price → plan
│   ├── queue :manapool  (1)    # API pushes (rate-limit friendly)
│   └── cron: catalog nightly 02:00 PT; prices daily 07:00 PT
└── MtgInventoryWeb.Endpoint
```

---

## 7. Data model

| Table | Key columns |
|---|---|
| `printings` | `id`, `scryfall_id` (unique), `mtgjson_uuid`, `name`, `set_code`, `collector_number`, `finish`, `rarity` |
| `variants` | `id`, `printing_id`, `condition`, `language`; unique (`printing_id`, `condition`, `language`) |
| `external_ids` | `variant_id`, `marketplace`, `external_id`; unique (`marketplace`, `external_id`) |
| `price_snapshots` | `variant_id`, `source`, `price_point` (`market`/`low`/`manapool_low`), `amount_cents`, `currency`, `listing_count`, `as_of`; index (`variant_id`, `price_point`, `as_of` desc) |
| `import_batches` | `id`, `filename`, `status` (`parsing`/`planned`/`applied`/`failed`), counts, timestamps |
| `batch_rows` | `batch_id`, `row_number`, `raw` (`jsonb`), `parse_errors`, `match_status`, `candidate_variant_ids`, `chosen_variant_id`, `excluded` |
| `plans` | `id`, `kind` (`import` now; `reprice`/`move` later), `source_batch_id`, `status`, `created_at`, `applied_at` |
| `plan_items` | `plan_id`, `variant_id`, `marketplace`, `quantity_delta`, `old_price_cents`, `new_price_cents`, `market_cents`, `reason`, `flags` (array), `status` (`ready`/`needs_review`/`excluded`/`applied`/`awaiting_confirmation`/`failed`), `result` (`jsonb`), `external_ref` |
| `listing_events` | `id`, `variant_id`, `marketplace`, `type` (`listed`/`repriced`/`quantity_changed`/`sold`), `quantity_delta`, `price_cents`, `occurred_at`, `plan_item_id`, `external_event_key` (unique, nullable) |
| `settings` | single row of rule parameters (§3) |

`listing_events` is append-only (no updates or deletes; enforced in the context API and with a DB trigger).

---

## 8. M1 flows

### 8.1 Catalog refresh (nightly)

1. Download MTGJSON identifiers (CSV variant where available) and `TcgplayerSkus`.
2. **Stream** both; upsert `printings`, `variants`, `external_ids`.
3. Record the MTGJSON build date. A failure keeps the previous catalog and surfaces a warning.

### 8.2 Price refresh (daily, and on demand)

1. Determine set codes in active listings ∪ pending batches.
2. `Prices.TCGTracking.fetch/2` per set (ETag-conditional, concurrency 1, identifying User-Agent with no personal data).
3. Persist snapshots. An import containing a set with no snapshot triggers an on-demand fetch for that set.
4. **Manual path:** the user uploads a TCGplayer *Export Filtered CSV*; parsed into `price_snapshots` with source `tcgplayer_export`.

### 8.3 Import → plan

1. User uploads a ManaBox CSV. Header check rejects non-ManaBox files before a batch is created.
2. Oban job `Imports.PlanBatch`:
   1. **Parse** each row into an embedded schema via changeset. Invalid rows → `needs_review` with errors.
   2. **Match:** Scryfall ID → printing (with finish) → variant (condition, language) → TCGplayer SKU. Result `exact` / `ambiguous` / `unmatched`.
   3. **Assign site** (§3.1) and **price** (§3.2 / §3.3) from resolved snapshots (§3.4).
   4. **Build plan items** with flags: `no_scan_price`, `unmatched`, `ambiguous`, `no_price`, `stale_price`, `thin_market`, `high_value`, `above_manapool_low`, `token`, `at_floor`, `special_attributes`.
3. Progress broadcasts over PubSub to the batch LiveView.

ManaBox → TCGplayer mappings: condition `near_mint`→`Near Mint`, `lightly_played`→`Lightly Played`, `moderately_played`→`Moderately Played`, `heavily_played`→`Heavily Played`, `damaged`→`Damaged`; language ISO code → MTGJSON language name; foil `normal`→`Normal`, `foil`/`etched`→`Foil` on the finish-appropriate product. Rows with `Proxy`, `Misprint`, `Altered` or `Signed` = true → `needs_review` (`special_attributes`). Tokens (MTGJSON token layout) are flagged `token`.

### 8.4 Review (LiveView)

- Summary counts by status and by site.
- Filterable list (status, site, flag). Per row: card name, set, number, finish, condition, quantity, assigned site, computed price with the inputs that produced it, flags.
- Actions: choose a candidate printing; override price; switch site; exclude; mark reviewed.
- Bulk actions: accept all `ready`; exclude all tokens.

### 8.5 Apply

- **Manapool (`:api`):** Oban job pushes `ready` items in chunks. Each item's `external_ref` = plan item ID (sent as `custom_external_id` if supported). Per-item results stored; failures retryable individually.
- **TCGplayer (`:file`):** generate a staged-import CSV mirroring the column set of a real *Export Filtered CSV* (exact headers confirmed in spike T-0), filling **TCGplayer Id** (SKU), **Add to Quantity** (= quantity), **TCG Marketplace Price**. Items → `awaiting_confirmation`. The user clicks **"I uploaded it"** to mark them `applied`. M2 later verifies against an export.
- **Record:** each `applied` item writes a `listed` event.

---

## 9. Error handling

| Failure | Behavior |
|---|---|
| Malformed row | Row → `needs_review` with reasons; batch continues |
| Wrong file type | Rejected at upload; no batch created |
| Unmatched / ambiguous | `needs_review` with candidates |
| Missing or stale price | `needs_review` / `stale_price`; banner if newest snapshot > 3 days |
| Guard breaches | Flagged, never auto-adjusted |
| Price source down | Oban retries with backoff; on exhaustion use last good snapshot and show its age |
| Manapool API error | Per-chunk retry; per-item status; "retry failed items" action; idempotent via `external_ref` |
| HTTP 429 | Job snoozes per `Retry-After` |
| Unexpected exception in a job | Job fails and is recorded by Oban; supervision keeps the app running |

---

## 10. Testing

- **Pure core:** unit tests plus StreamData properties, including:
  - TCGplayer price ≥ floor, ≥ round_down(market × 0.85), and ≤ low when `low × 0.99` is the max candidate.
  - Site assignment is total and deterministic; exactly $3.00 → Manapool.
  - Building a plan twice from the same inputs is identical.
- **Matching regression fixture:** the user's 186-row ManaBox export (`test/fixtures/manabox/near_mint_2026-11-01.csv`) with expected matches, covering promos (`176p`), star promos (`185★`), The List (`MOR-118`), etched foils, double-faced cards, tokens without prices, and duplicate names across sets.
- **Adapters:** `Req.Test` stubs with recorded real responses; separate manually run contract tests (`mix test --only contract`) against live services.
- **LiveView:** upload → flagged rows → resolve → apply, with `Phoenix.LiveViewTest`.
- **Database:** Postgres via Ecto SQL sandbox.

---

## 11. Engineering practices

- GitHub Issues per plan task, grouped under an **M1** milestone. One PR per ticket.
- CI on every PR: `mix format --check-formatted`, `mix credo --strict`, `mix dialyzer`, `mix test` with a Postgres service, coverage report.
- `main` is protected: green CI + owner review required.
- Secrets (Manapool token) via environment variables (`.env` git-ignored); never committed.
- ADRs in `docs/adr/`:
  1. Elixir + Phoenix LiveView
  2. Postgres in Docker from day one
  3. Plan/apply for all outbound changes
  4. Append-only listing record; sites own quantity
  5. Money as integer cents
  6. TCGplayer as file transport (no API)
  7. TCGTracking as primary price source with fallbacks

---

## 12. Spike T-0 (first ticket) — verify before building on assumptions

1. **Manapool API:** endpoints to create/update listings; accepted identifiers (Scryfall ID vs TCGplayer SKU vs Manapool ID); bulk support; `custom_external_id`; rate limits; whether it exposes Manapool lows.
2. **TCGTracking:** live calls for 3 sets from the fixture; confirm SKU IDs match MTGJSON `skuId`; confirm `mp` coverage; email maintainer re: personal use.
3. **TCGplayer CSV:** user exports one *Export Filtered CSV* and one *Export from Live*; record exact headers into fixtures.
4. **MTGJSON:** confirm file sizes, CSV identifier availability, and SKU coverage for the fixture's 186 rows.

Outcome: a short findings note in `docs/spikes/` and, if any assumption fails, an amendment to this spec before M1 implementation continues.

---

## 13. Risks

| Risk | Mitigation |
|---|---|
| TCGTracking changes or disappears | Fallback chain; manual TCGplayer export path always works |
| MTGJSON SKU mapping gaps | Review queue; candidate selection persists as a manual mapping |
| Manapool API differs from assumptions | Spike T-0; ManaBox-format CSV fallback |
| TCGplayer staged-import format changes | Headers kept in one module + fixture tests from real exports |
| Learning curve (Elixir/OTP/LiveView) | Small tickets; pure core first; ADRs capture reasoning |
| Repo is public | Pricing rules are visible to anyone; no secrets or sales data committed. Owner may choose to make the repo private. |

---

## 14. Out of scope for M1

Repricing jobs, reconciliation, cross-site moves, sales tracking/charts, authentication, hosting, CardTrader/eBay adapters, cosmetic price rounding, and syncing quantities back from the sites.
