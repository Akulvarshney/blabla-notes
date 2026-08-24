# RAG + Calculation-Backend Chatbot Architecture
### Excel-only data source (pandas), ChromaDB as vector store — no SQL DB anywhere

---

## 1. Core design principle (unchanged, adapted to Excel)

Same split as before, just with Excel/pandas standing in for the "structured DB":

| Data type | Where it lives | How it's queried |
|---|---|---|
| Numeric / categorical / scores / flags (90% of columns) | **In-memory pandas DataFrame**, loaded from the Excel file | pandas `groupby`, `query`, `sort_values`, `nlargest` — **never sent raw to the LLM** |
| Free-text narrative fields | **ChromaDB** collection(s), with metadata filters | Semantic similarity search filtered by `masked_mpxn` / segment / cluster / campaign_id |
| Business glossary (cluster definitions, grading thresholds) | Small ChromaDB collection | Semantic search for definitional questions |
| Conversation history | In-memory dict / JSON file per session (or Redis if you want persistence across restarts) | Rolling window + periodic summarization — no SQL needed, it's just key-value |

No SQL engine is introduced anywhere — pandas replaces the "query layer," and
Chroma replaces the "search layer." This holds even as the dataset grows; see
§3 for how to keep it fast without a database server.

---

## 2. High-level flow (same shape, different engine underneath)

```
User question
     │
     ▼
[1] Memory Fetch ─── last k turns + entity memory (active customer/segment/campaign)
     │
     ▼
[2] Intent & Slot Extraction (LLM call #1, JSON/function-calling mode)
     │   → query_type: computation | explanation | hybrid | generative
     │   → entities/filters/metric/groupby/sort/limit
     │
     ▼
[3] Router
     ├── Computation ──► Dynamic Query Engine (filter/derive/aggregate on the in-memory df)
     ├── Explanation ──► ChromaDB similarity search (metadata-filtered)
     ├── Hybrid ───────► both, combined
     └── Generative ───► structured facts + Chroma snippet → LLM drafts new text
     │
     ▼
[4] Context Assembly → { question, calc_result, retrieved_snippets, memory_summary }
     │
     ▼
[5] Final Generation (LLM call #2) — "use only the numbers given, don't compute"
     │
     ▼
[6] Response to user  →  [7] Memory Update
```

---

## 3. Data layer: Excel → pandas, kept fast without a DB

1. **Load once at startup** (or on file-change) with `pandas.read_excel` /
   `openpyxl` engine. For a *huge* file, `read_excel` itself can be slow — the
   fix is not a database, it's a faster on-disk cache format:
   - After the first load, write the DataFrame to **Parquet** (`df.to_parquet()`)
     in a local `/cache` folder. Parquet is just a file format (columnar,
     compressed) — it is **not** a database. On subsequent app restarts, load
     from Parquet (milliseconds–seconds) instead of re-parsing the Excel file
     (which can take minutes on a wide, multi-thousand-row sheet).
   - If the underlying Excel changes, re-run the loader → re-derive helper
     columns → re-write the Parquet cache. This can be a manual trigger
     ("refresh data") or a file-watcher (e.g. `watchdog` on the `.xlsx` path).
   - Optional: if pandas itself feels slow on the full sheet, swap in **Polars**
     (same idea, much faster on wide numeric data, still just a DataFrame
     library, still no server/DB involved).
2. **Derive helper columns once at load time**, not per-query — store them as
   extra columns in the same in-memory DataFrame / Parquet cache:
   - `tariff_health_grade` bucketed from `tariff_efficiency_score` (confirm the
     bucketing rule with the data owner if this column doesn't already exist).
   - Precomputed peak-hour usage % from `hh1`–`hh48` for cluster-explanation
     questions (Q4).
   - Portfolio-level rollups (segment counts, avg scores) computed once and
     cached as a small secondary DataFrame, refreshed alongside the main load —
     avoids recomputing a full `groupby` over the whole sheet on every chat turn.
3. **Access control**: a simple `advisor_id → [masked_mpxn]` mapping — this can
   itself just be another sheet/tab in the same Excel file, or a small JSON/CSV
   config file loaded into a Python dict at startup. The query engine
   filters the DataFrame down to the caller's portfolio *first*, before any
   filter/derive/aggregate step runs.
4. **Text corpus extraction for embeddings** — same fields as before
   (`ai_customer_narrative`, `propensity_signals`, `rec_*_reason`,
   `campaign_message`, cluster-glossary write-ups), pulled out of the DataFrame
   and pushed into Chroma (see §5). Raw numeric columns are still never embedded
   as text — that's the mistake that causes the LLM to eyeball numbers instead
   of computing them.

---

## 4. Calculation Engine — one dynamic query engine, not a fixed function menu

Earlier drafts of this architecture used a handful of named functions
(`get_top_n`, `get_count`, `get_groupby_agg`...). That's an unnecessary
restriction — filtering, deriving new columns, and aggregating are a closed,
universal set of operations for tabular analysis, so there's no reason to
enumerate "question types" in advance. The actual design is:

- **One** generic engine, `run_dynamic_query(df, spec)`, parameterized by:
  `filter_expr` (a boolean expression over existing columns), `derived_columns`
  (new columns computed from existing ones — ratios, sums, differences),
  `group_by`, `metric`, `agg` (`sum`/`mean`/`count`/`min`/`max`/`std`/`median`/
  `nunique`/`corr`), `sort_by`/`sort_desc`, `limit`, `select_columns`.
- The LLM is shown the **live column list** of the DataFrame on *every*
  request (read straight off `df.columns`, not hardcoded in a prompt), and
  builds the query spec fresh each time. If a new column appears in the Excel
  tomorrow, it's usable immediately — no prompt or code change needed.
- New questions need new *parameter combinations*, not new Python functions.
  A question like "what's the ratio of missed savings to bill size, by
  region" is just a `derived_columns` entry plus a `group_by` — not a reason
  to write a 7th named function.
- **Safety**: `filter_expr` and every `derived_columns.expr` are validated
  against an AST whitelist before they ever reach pandas — only arithmetic,
  comparisons, and boolean logic on known column names are permitted. No
  function calls, attribute access, imports, or comprehensions can appear.
  This is what makes "dynamic" safe: the LLM can build any filter/aggregate
  combination it wants, but it cannot execute arbitrary code.
- **Row-level security** (advisor → portfolio scoping) is applied to the
  DataFrame *before* the query spec runs, in code — never left to the LLM to
  remember.
- **capability_gap**: the one thing this engine can't express is genuinely
  new *kinds* of computation — multi-period trend analysis, what-if
  simulation, writing data back. When the planner hits one of these, it sets
  a `capability_gap` flag with a reason instead of guessing; these get logged
  to a file for periodic review. That log is the actual answer to "who adds
  new capability, and when" — you review it, spot the recurring patterns, and
  only then does a new capability get engineered (rare, and now backed by
  real evidence of demand rather than a guess).

---

## 4a. Formula Engine — named business formulas, backend-computed only

Separate from the dynamic query engine in §4, because some calculations
aren't a generic "filter + aggregate" — they're specific formulas the
business already owns (e.g. a bespoke blended efficiency index, a
risk-adjusted propensity score). These need to be defined once in code and
never recomputed on the fly by an LLM guessing at the formula's shape.

**Registry — a static config, not something the LLM generates:**

```python
FORMULA_REGISTRY = {
    "blended_efficiency_index": {
        "description": "Weighted efficiency score combining tariff_efficiency_score and peak_usage_ratio",
        "required_columns": ["tariff_efficiency_score", "peak_usage_ratio"],
        "params": {"weight_efficiency": 0.6, "weight_peak": 0.4},  # overridable within bounds, optional
        "fn": blended_efficiency_index,      # plain pandas function, vectorized
        "output_field": "blended_efficiency_index",
    },
    "risk_adjusted_propensity": {
        "description": "Switch propensity discounted by contract lock-in months remaining",
        "required_columns": ["switch_propensity_score", "contract_months_remaining"],
        "params": {},
        "fn": risk_adjusted_propensity,
        "output_field": "risk_adjusted_propensity",
    },
    # ... one entry per business formula
}
```

Each `fn` is ordinary vectorized pandas code — the arithmetic never touches
the LLM.

**What the LLM decides (same intent-extraction call as today):**
- `formula_id` — which registered formula applies, chosen from a catalog of
  `{id, description, required_columns}` shown to the LLM alongside the live
  column list (same pattern as §4's live-column-list trick)
- `filter_expr` — which customers it applies to (the *same* AST-whitelisted
  expression syntax as §4 — "grade D/E only", "this portfolio",
  `masked_mpxn == 'X'`, etc.)
- `param_overrides` — only for params the registry marks overridable, only
  within registry-defined bounds

The LLM never writes formula logic and never produces the numeric result —
it only names a `formula_id` and a target population, the same way it
already names a `query_type` today.

**Backend execution order (mirrors §4's security-first pattern):**
1. Reject an unrecognized `formula_id` outright → log it as a
   `capability_gap` (same mechanism as §4), don't substitute a guess.
2. Check every entry in `required_columns` actually exists in the *current*
   `df.columns` — if an Excel refresh dropped or renamed a column a formula
   depends on, that's also a `capability_gap`/data-gap event, never a silent
   fallback to a nearby column.
3. Apply row-level security (advisor → portfolio scope) to the DataFrame
   first, same as every other path.
4. Apply `filter_expr` (AST-validated, same whitelist as §4) to narrow down
   to the target customers.
5. Run `fn(filtered_df, **merged_params)` — vectorized, not a per-row Python
   loop, wherever the formula allows it.
6. Return `{formula_id, params_used, masked_mpxn: value, ...}` as part of
   `calc_result`, so Context Assembly (§1 step 4) carries the formula name
   forward and the final-generation prompt can say *"using
   <formula_id>"* rather than presenting a bare number with no provenance.

**Router (§2) update:** add a `Formula` branch alongside
`Computation`/`Explanation`/`Hybrid`/`Generative` — or fold it into
`Computation` as a sub-mode (`calc_mode: dynamic | formula`) if the two tend
to be requested in the same turn (e.g. "run the blended efficiency index for
grade D/E customers, and also show me their raw peak_usage_ratio" is
genuinely both dynamic and formula-based at once).

---

## 5. ChromaDB as the vector layer

Chroma fits this well since it's embedded/local (no server to run, persists to
disk) — matches the "no DB" constraint nicely.

**Setup:**
```python
import chromadb
client = chromadb.PersistentClient(path="./chroma_store")
collection = client.get_or_create_collection("customer_narratives")
```

**Collection design** — one collection per doc type keeps filtering simple and
fast, and matches how you'll query them:
- `customer_narratives` — one doc per customer (`ai_customer_narrative` +
  `propensity_signals` text + top rec reasons), metadata: `masked_mpxn`,
  `region`, `segment`, `cluster`, `tariff`
- `cluster_glossary` — one doc per behaviour/consumption cluster type, written
  once, rarely changes
- `campaign_docs` — one doc per campaign (`campaign_message`, `campaign_action`),
  metadata: `campaign_id`, `channel`, `date`

**Ingest / upsert** (re-run whenever the Excel refreshes — use `upsert` keyed by
a stable ID so re-embedding is idempotent and cheap, only changed docs get
re-embedded if you diff against the previous run):
```python
collection.upsert(
    ids=[masked_mpxn_1, masked_mpxn_2, ...],
    documents=[narrative_text_1, narrative_text_2, ...],
    metadatas=[{"region": "...", "cluster": "...", "segment": "..."}, ...],
)
```

**Query patterns:**
- *Semantic + filtered* (Q4's "why this cluster"): 
  `collection.query(query_texts=["why evening peak concentrator"], where={"masked_mpxn": "XYZ"}, n_results=3)`
  — Chroma's `where` clause does the metadata filtering, so you're not doing a
  blind similarity search across the whole customer base.
- *Direct lookup, no similarity search needed* (Q8, Q9 — when you already know
  the exact `masked_mpxn`/`campaign_id`): just call `collection.get(ids=[...])`
  instead of `.query()`. This skips embedding the query entirely and is faster
  and more precise than treating a known-ID lookup as a semantic search.

---

## 6. Conversation memory (no SQL, unchanged philosophy)

- **Short-term**: last 4–6 turns kept in a per-session in-memory dict (or a
  small JSON file per session if you want it to survive a process restart) —
  no database needed for this either.
- **Entity memory**: `{active_customer, active_segment, active_campaign_id}` —
  a plain dict, updated after each turn, used to resolve "this customer" /
  "that segment" in the next intent-extraction call.
- **Rolling summary**: once history passes ~6 turns, an LLM call compresses
  older turns into 2–3 sentences and the raw turns are dropped — keeps token
  cost flat regardless of session length.
- If you need this to survive app restarts / scale across multiple workers,
  **Redis** is a reasonable add (key-value, not SQL) — optional, not required
  for a single-process deployment.

---

## 7. Walking through your 9 example questions (Excel/pandas/Chroma version)

| # | Question | Type | Query spec the planner builds | Chroma used? | LLM's job |
|---|---|---|---|---|---|
| 1 | Most likely to switch this month | Computation | `sort_by=switch_propensity_score, sort_desc=true, limit=20, select_columns=[masked_mpxn, switch_propensity_score, propensity_band]` | No | Format ranked list, summarize common `propensity_signals` |
| 2 | Count of grade D/E | Computation | `filter_expr="tariff_health_grade in ['D','E']", metric=masked_mpxn, agg=count` | No | State the number + % of portfolio |
| 3 | Segments with highest savings opportunity | Computation | `group_by=[consumption_cluster], metric=missed_saving_gbp, agg=sum, sort_desc=true` | Optional — pull that segment's glossary blurb | Narrate top segments, magnitude, suggested action |
| 4 | Why cluster assignment for a customer | Hybrid | `filter_expr="masked_mpxn=='X'", derived_columns=[{evening_peak_pct: sum(hh34..hh42)/sum(hh1..hh48)}]` | **Yes** — direct lookup on `customer_narratives` + `cluster_glossary` search | Combine computed % with narrative to explain plainly |
| 5 | Main consumption patterns in "my portfolio" | Computation (portfolio-scoped) | `group_by=[behaviour_cluster], metric=masked_mpxn, agg=count` | Optional — glossary docs for plain-language cluster descriptions | Turn counts into a readable summary |
| 6 | Suitable for time-of-use tariff | Computation | `filter_expr="eligibility_status=='eligible'", sort_by=tou_readiness_score, sort_desc=true, limit=20` | No | List candidates + `estimated_peak_reduction_pct` as rationale |
| 7 | Opportunity by tariff/region/segment | Computation | `group_by=[tariff, region], metric=missed_saving_gbp, agg=sum` (pivot) | No | Present as table, call out top 2–3 |
| 8 | Who to target next campaign, and why | Hybrid, multi-criteria | `derived_columns=[{composite_score: weighted expr}], sort_by=composite_score, limit=15` | **Yes**, direct `.get(ids=[...])` lookup, not similarity search | Explain "why" per customer, tie to score drivers |
| 9 | What happened with campaign X, follow-up? | Hybrid + Generative | `filter_expr="campaign_id=='X'", group_by=[campaign_channel], metric=masked_mpxn, agg=count` | Yes — `campaign_docs.get(ids=[campaign_id])` | Backend rule decides *if* follow-up warranted; LLM only **drafts** the copy |

None of these needed a bespoke function — every row is the same generic
engine with different parameters. That's the point: tomorrow's unseen
question is very likely just another combination of the same primitives.

---

## 8. Guardrails (same as before, restated for this stack)

- **Numeric grounding**: final-generation prompt states *"only reference
  numbers present in CALC_RESULT; if something isn't there, say the data isn't
  available."*
- **Row-level security**: enforced by the query engine itself (filter by
  portfolio before anything else), not left as a prompt instruction.
- **Expression safety**: every `filter_expr` and `derived_columns.expr` is
  checked against an AST whitelist (arithmetic/comparisons/boolean logic on
  known columns only) before it touches pandas — this is what makes an
  open-ended query interface safe rather than "run whatever the LLM writes."
- **Re-embedding cost control**: only upsert to Chroma the customers/campaigns
  whose text fields actually changed since the last Excel refresh (diff by
  hashing the text or comparing a `last_modified`/version column), rather than
  re-embedding the whole file every time.
- **PII**: `masked_mpxn` is already masked — make sure narrative text pulled
  into Chroma doesn't contain any unmasked identifiers before it's embedded.
- **Formula provenance**: whenever `calc_result` comes from the Formula
  Engine (§4a), the final-generation prompt must state which `formula_id`
  produced the number — the LLM narrates the named formula's output, it
  never recomputes or approximates a value a registered formula already
  produced.

---

## 9. Stack summary

| Layer | Tool |
|---|---|
| Structured data | pandas (or Polars) DataFrame, loaded from Excel, cached as Parquet |
| Vector store | ChromaDB (`PersistentClient`, local disk) |
| Embeddings | Any current embedding model (local sentence-transformer or hosted API) |
| Orchestration | FastAPI service, custom routing logic (framework optional) |
| Memory | In-memory dict / JSON per session; Redis optional for persistence |
| LLM calls | Fast/cheap model for intent-extraction (structured output), stronger model for final generation |

---

## 10. Open items to confirm

- Is `tariff_health_grade` an actual column, or derived from
  `tariff_efficiency_score`? (Q2 assumes a bucketing rule.)
- What columns capture campaign **outcomes** (open/response/conversion)? Needed
  for Q9 and not visible in the current column brief.
- How often does the Excel file actually get updated — daily batch, ad hoc
  export? This determines whether the Parquet-cache-refresh should be a manual
  trigger or a scheduled/file-watch job.
- What's the actual list of business formulas for §4a's registry, and for
  each one: which columns does it read, is it per-customer or
  per-group/aggregate, and are any of its parameters meant to be
  user-adjustable (vs. fixed constants)?
