# CLAUDE.md — MTG Inventory Manager

Local Phoenix LiveView app that turns ManaBox scans into priced, reviewed listings on **Manapool** (cards ≥ $3) and **TCGplayer** (cards < $3), and grows into reconciliation, repricing, cross-site moves and sales tracking. Single user. Built to production standards as an FP/Elixir learning project.

## Source of truth

Read these before changing behavior. If code and docs disagree, stop and ask.

- **Design spec:** `docs/superpowers/specs/2026-11-07-m1-scan-to-listings-design.md`. Business rules (§3), architecture (§6), data model (§7), flows (§8).
- **ADRs:** `docs/adr/`. Accepted decisions; supersede with a new ADR, never edit an accepted one.
- **Spike notes:** `docs/spikes/`. Verified facts about external systems.
- **Phoenix `AGENTS.md`** (from the generator), if present: framework usage rules. Follow it.

Business-rule numbers (0.99, 0.85, $0.15, 1.05/1.02/0.99, 15%, $50, $3.00, $2.50/$3.50) live in `settings`. Never hard-code them outside defaults and tests.

## Workflow

- **Every change starts from a GitHub Issue.** No issue, no branch.
- **Branches:** `<type>/<issue#>-<slug>`, e.g. `feat/12-tcgplayer-pricing-rule`. Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `spike`.
- **Commits:** Conventional Commits (`feat(pricing): add thin-market flag`). Small and focused.
- **One PR per issue.** Description links the issue (`Closes #12`), says what changed and why, and how it was tested. Keep PRs small enough to review in one sitting.
- **Never push to `main`.** CI must be green and the owner must approve.
- **Spec drift:** if implementation shows the spec is wrong, update the spec (or add an ADR) **in the same PR** and call it out in the description.
- **Spikes** produce a note in `docs/spikes/`, not production code. Spike code is throwaway.

## Commands

```bash
docker compose up -d db          # Postgres
mix setup                        # deps, DB create/migrate, assets
mix phx.server                   # http://localhost:4000
mix test                         # full suite (no network)
mix test --only contract         # live external APIs, run manually
mix format && mix credo --strict && mix dialyzer
mix precommit                    # run before every push
```

## Architecture principles

1. **Functional core, imperative shell.** Matching, site assignment, pricing and plan building are **pure functions over plain data**: no Repo, HTTP, time or randomness inside them. Pass `now`, settings and snapshots in as arguments. Effects live in adapters, Oban workers and context write functions.
2. **Plan, then apply.** Every outbound change (listings, reprices, moves) is first a `plan` of `plan_items` (old value, new value, market, reason, flags), reviewed in the UI, then applied. Never write to a marketplace outside the apply step.
3. **Sites own quantity; we own history.** `listing_events` is append-only: never update or delete rows. Current state is derived.
4. **Idempotent by design.** Outbound pushes carry `external_ref` (the plan item ID). Inbound events carry a unique `external_event_key`. Any Oban job must be safe to run twice.
5. **Flag, don't silently fix.** Guard breaches (high value, above Manapool low, thin market, stale price, ambiguous match) become flags for review, never automatic adjustments.
6. **Context boundaries.** `Catalog`, `Prices`, `Imports`, `Matching`, `Pricing`, `Plans`, `Listings`, `Marketplaces`, `Settings`. Call other contexts only through their public functions; never query another context's schemas directly. Web modules call contexts, never `Repo`.
7. **Adapters behind behaviours.** Marketplaces implement `Marketplaces.Marketplace` (transport `:api` or `:file`). Price providers implement `Prices.PriceSource`. New integrations = new adapter, no changes to the core.
8. **YAGNI.** Build only what the current milestone's spec requires. No speculative abstraction for marketplaces we don't integrate yet.

## Elixir coding standards

- **Money:** integer cents + currency code. Intermediate math with `Decimal`. **No floats for money, ever.** Rounding direction is part of the rule (TCGplayer rounds down, Manapool rounds half-up). Name it explicitly.
- **Return values:** `{:ok, value} | {:error, reason}` for fallible functions. Use `with` for pipelines of fallible steps. Raise only for programmer errors. Bang functions (`foo!/1`) only where crashing is the right outcome.
- **Pattern match in function heads** over conditionals. Prefer multiple clauses to nested `if`/`case`. Make impossible states unrepresentable with structs and tagged tuples.
- **Pipelines** start with a value, not a function call, and stay readable (≤ ~6 steps or extract a named function).
- **Typespecs** on every public function; `@moduledoc` on every public module; `@doc` on public functions in contexts and behaviours. Dialyzer must pass with no new warnings.
- **Structs over maps** for domain data crossing module boundaries. Use `@enforce_keys`.
- **Ecto:** validate all external input (CSV rows, API payloads) through changesets or embedded schemas. Constraints in migrations as well as changesets (unique indexes, NOT NULL, check constraints). Migrations are forward-only once merged.
- **Oban workers:** small, idempotent, and delegate to context functions. Use `unique` options where duplicates are possible. Return `{:snooze, seconds}` on 429 / `Retry-After`. Record per-item results, not just job success.
- **HTTP:** Req only. One client module per external service, with base URL, headers (identifying User-Agent, no personal data), retry and timeout config. Respect ETags and rate limits; concurrency 1 for third-party APIs.
- **LiveView:** keep business logic out of LiveViews; they call contexts and render. Use streams for long lists. Long work runs in Oban and reports progress over PubSub, never in `handle_event`.
- **Naming:** domain language from the spec (`printing`, `variant`, `plan_item`, `snapshot`, `flag`). No abbreviations except well-known ones (`id`, `sku`).
- **No dead code, no commented-out code, no `IO.inspect`** in committed code. Use `Logger` with metadata.

## Testing standards

- **TDD for the pure core.** Write the failing test first for matching, pricing, site assignment and plan building.
- **Property tests (StreamData)** for invariants, e.g. TCGplayer price ≥ floor and ≥ round_down(market × 0.85); exactly $3.00 → Manapool; plans are deterministic.
- **Regression fixture:** `test/fixtures/manabox/near_mint_2026-11-01.csv` (real 186-row scan). Matching changes must keep its expected results passing; update expectations deliberately and explain in the PR.
- **No network in `mix test`.** Stub with `Req.Test` using recorded real responses in `test/fixtures/`. Live checks are tagged `@tag :contract` and excluded by default.
- **LiveView flows** tested with `Phoenix.LiveViewTest`. **DB** via Ecto SQL sandbox (async where possible).
- Every bug fix ships with a test that fails without the fix.
- Test names describe behavior (`"prices at floor when market × 0.85 and low × 0.99 are below $0.15"`), not implementation.

## Security and data handling

- **Secrets** (Manapool token, etc.) only via environment variables / git-ignored `.env`. Never commit tokens, real order data, or buyer information. **The repo is public.**
- **No scraping.** Data comes from official APIs, official seller-portal exports, or open data providers (MTGJSON, TCGTracking). Don't add any integration that violates a site's terms.
- Treat all external data (CSV uploads, API responses) as untrusted: validate, never `String.to_atom/1` on input, never interpolate into SQL.

## Docs

- New architectural decision → new ADR (`docs/adr/NNNN-title.md`: Context, Decision, Consequences).
- User-visible behavior change → update the spec section it touches.
- Keep this file current. If a convention here is wrong or missing, fix it in a PR.

## When working as an AI assistant here

- Read the relevant spec section and ADRs before proposing a change; cite them in the PR.
- Ask before: changing business-rule semantics, adding a dependency, adding an external integration, or changing the data model beyond the ticket.
- Prefer the smallest change that satisfies the ticket. Don't refactor unrelated code in the same PR.
- Run `mix precommit` and report the actual results; never claim tests pass without running them.
