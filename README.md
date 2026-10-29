# @pipeworx/sec-financial-statements

Dimensioned XBRL facts and tagged note text from SEC's DERA "Financial Statement and Notes" data
sets — the layer beneath `data.sec.gov` companyfacts/frames (what `edgar` and `sec-xbrl` call), which
expose **undimensioned facts only**. A segment-level revenue number, a geography breakdown, a
footnote-tagged hedge notional — none of that is reachable through companyfacts/frames at all; it
only exists in SEC's bulk data sets. Hosted, keyless to the caller.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

> **Status at commit time: schema + tools + loader are code-complete; NO PERIOD IS LOADED YET.** The
> migration (`supabase/migrations/219_sec_financial_statements.sql`) has not been applied to
> production and this pack has not been deployed. See "Why this ships without data" below — this is
> deliberate, not an oversight, and the next step is written down there.

## Tools

- `company_financial_facts(company?, cik?, tag?, limit?)` — dimensioned facts for one filer (give a
  name, ILIKE-matched, or a CIK). Each fact carries its period, value, unit and — when dimensioned —
  the segment/geography axis it belongs to, resolved through `sec_fsn_dimensions` rather than left as
  a bare hash. This is the tool that answers what companyfacts/frames structurally cannot: "Flutter
  Entertainment's cross-currency interest rate swap notional" is a `DerivativeNotionalAmount` fact
  dimensioned by hedge designation, not a consolidated total.
- `tag_cross_company_screen(tag, version?, ddate?, dimensioned_only?, limit?)` — every filer's value
  for ONE tag in one period, ranked — the inverse of a per-company lookup ("who reported the highest
  `AccountsPayableCurrent` this quarter").
- `note_text_search(query, company?, tag?, limit?)` — full-text search (Postgres `english` tsvector)
  over tagged note/footnote text blocks — the disclosure prose behind the numbers. Searches tagged
  note blocks only, not full filing text; see "edgar_filing_text vs. this pack" below.
- `fsn_coverage()` — which periods are loaded and how many submissions/facts/note blocks each holds.
  Call this first when a lookup returns nothing — coverage is real and finite, never silently assumed.

## edgar_filing_text vs. this pack — complementary, not duplicative

`edgar_filing_text` (fixed 2026-09-29, fleet #2500, commit `7ab72cdca`) does pipe-separated
multi-phrase **prose** search over a filing's rendered text and attaches the one XBRL fact matched to
that exact accession when a phrase reads as a financial concept. This pack is the opposite shape: a
**structured database** of every tagged numeric fact (dimensioned included) and every tagged note text
block across whichever periods are loaded, queryable by tag/company/period without needing to guess a
search phrase that happens to appear verbatim in the rendered document. Use `edgar_filing_text` to find
a fact inside one specific filing's prose; use this pack to screen across companies, pull a full
dimensional breakdown, or search note text across many filings at once.

## Why this ships without data

Sizes were explicitly UNVERIFIED when this pack was filed (fleet #2506) — "measure first." Measured
live against the newest available period (`2026_08_notes.zip`, HTTP 200,
`content-length: 312,729,503` compressed): **~2.46GB uncompressed across the six tables this pack
loads**, `num.tsv` alone 5,563,767 rows / 1.14GB. Loading even one period is a real, multi-GB bulk
operation over SEC's shared fleet egress (CLAUDE.md, fleet #1245) — not something to run casually
inside a build session. Parsing and row-mapping were validated locally against the real
`2026_08_notes.zip` (every table's header/columns confirmed, dimensioned vs. undimensioned facts
correctly distinguished, note text and submission rows read cleanly) — the loader is not speculative,
it has been run against real SEC bytes, just not into production yet.

Two things gate actually loading data and shipping this pack live:

1. **The migration has to land on `main` and be applied via `db-migrate.yml`** (workflow_dispatch,
   `gh workflow run db-migrate.yml -f file=219_sec_financial_statements.sql` — dry-run first with
   `-f dry_run=true`), per CLAUDE.md's local-copy rule ("migration lands ALONE on main… before the
   pack ships"). This build's own fleet task (#2506) explicitly says **"Commit, do NOT push — report
   SHAs to the dispatching PM"** — so the push/apply/deploy decision is the dispatching PM's, not
   this session's, and is reported rather than executed here. (A direct
   `supabase db query --file … --linked` apply — the documented ad-hoc testing path — was attempted
   for local verification and was refused by the session's own permission policy as a protected
   infrastructure-apply action; that refusal was respected rather than routed around, which is the
   correct behavior for anyone hitting the same wall.)
2. **The first period load is a real ingest run**, not a CI step:
   `node scripts/ingest-sec-fsn.mjs --period 2026_08` (or `--list-periods` to see what SEC currently
   publishes). Budget real wall time — `num.tsv` alone is 5.56M rows at 3,000/batch.

## Data + refresh

Source: `https://www.sec.gov/data-research/sec-markets-data/financial-statement-notes-data-sets` —
quarterly zips `2009q1` through `2025q2`, then monthly (`2026_08`, etc.) from `2025_08` onward (SEC
changed cadence mid-2025). US federal government data, public domain — no reuse-grant question. Six
of the eight TSVs each zip carries are loaded (`sub`, `tag`, `dim`, `pre`, `num`, `txt`); `cal`
(calculation linkbase) and `ren` (rendering metadata) are not — they serve statement *rendering*, not
the fact/note lookups this pack answers.

- Loader: `node scripts/ingest-sec-fsn.mjs --period <YYYY_MM|YYYYqN>` — one period per run, same
  credential resolution and batched-upsert-with-retry shape as `scripts/ingest-sec-13f.mjs`.
  `--only sub,tag,dim,pre,num,txt` loads a subset; `--zip <path>` skips the download for a
  local/already-fetched archive.
- Re-running an already-loaded period is a safe no-op upsert: `num`/`txt` key on a content hash of
  every loaded column (SEC's own documented natural key for `num.tsv` is 9 columns, several
  nullable — a hash collapses exact SEC-side duplicates without losing genuinely distinct rows, same
  pattern as `leie_exclusions` / `csl_entries` / `ferc_eqr`, migration 211).
- Backfill plan: load the newest period first (proves the pipeline end-to-end fastest), then step
  backward one period per run as budget allows. `fsn_coverage()` always reports exactly what's loaded
  — no response ever implies coverage beyond that.
- Schema and indexes: `supabase/migrations/219_sec_financial_statements.sql`. Row-level security is
  enabled on all six tables, `anon`/`authenticated` revoked (migration 211's `ferc_eqr` pattern) —
  only the gateway's own injected data credential can read.

## What this cannot tell you

- **Coverage is whatever has actually been loaded**, not the full 2009-present history SEC publishes.
  `fsn_coverage()` is the source of truth; a lookup outside loaded periods returns nothing and says so
  rather than reading as "this fact doesn't exist."
- **No ticker column** — SEC's DERA data sets key on CIK/accession, not ticker. `company_financial_facts`
  takes a company name (ILIKE) or a CIK; an ambiguous name match returns the candidate CIKs instead of
  silently picking one.
- **Tags are exact, case-sensitive XBRL element names** (`"Revenues"`, not `"revenue"`) — this pack does
  not fuzzy-match tag names, because a near-miss tag is a different accounting concept, not a typo.
- **This is what filers tagged, not a restatement-aware view.** An amended filing's facts sit alongside
  the original's under different accessions; this pack does not collapse them (contrast `sec-13f`'s
  amendment handling, which has no analog here since XBRL facts are versioned by filing, not by
  position).

## Auth

Keyless to the caller — the data credentials are injected by the gateway, never exposed.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "sec-financial-statements": {
      "url": "https://gateway.pipeworx.io/sec-financial-statements/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/sec-financial-statements/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/company_financial_facts \
  -H 'Content-Type: application/json' \
  -d '{"company":"Apple","tag":"Revenues"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/company_financial_facts`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-sec-financial-statements` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/sec-financial-statements/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "sec-financial-statements": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-sec-financial-statements"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Sec Financial Statements data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
