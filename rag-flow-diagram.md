# RAG + Calculation-Backend Chatbot — Flow

Excel/pandas as the structured store · ChromaDB as the vector store · no SQL anywhere

## How to read this

The diagram has two halves. **Data layer** (top box) happens once, at startup
or whenever the Excel file is refreshed — it's where the `.xlsx` file is
loaded, split into a numeric/categorical DataFrame (cached to Parquet) and a
free-text stream (embedded into ChromaDB), and never touched again until the
next refresh. **Request flow** (bottom box) happens once per user turn — it's
the actual chat request path: intent extraction figures out what's being
asked, the router sends it to whichever engine can answer it (dynamic query,
formula, semantic search, or a hybrid/generative combination), and the
results get assembled and narrated back. The two dashed arrows crossing from
the data layer into the request flow show that the query and formula engines
never re-read Excel — they operate on the DataFrame that's already sitting in
memory.

```mermaid
flowchart TB

    subgraph DATA["DATA LAYER — built once, refreshed on Excel change"]
        EXCEL["📗 Excel file (.xlsx)<br/>source of truth<br/>incl. advisor→portfolio map"]
        LOAD["Load: pandas.read_excel()<br/>(first run / manual refresh)"]
        PARQUET["Parquet cache (disk)<br/>reload in ms on restart"]
        DF["🟧 In-memory DataFrame<br/>numeric/categorical + derived cols<br/>(VECTORIZED — pandas ops)"]
        TEXT["Free-text columns<br/>(narrative, rec_reason, campaign_msg)"]
        EMBED["Embedding model"]
        CHROMA["ChromaDB (local disk)<br/>customer_narratives · cluster_glossary · campaign_docs"]

        EXCEL --> LOAD
        LOAD --> PARQUET --> DF
        LOAD --> DF
        LOAD --> TEXT --> EMBED --> CHROMA
    end

    subgraph FLOW["REQUEST FLOW — runs once per user turn"]
        Q["User question"]
        MEM["Memory fetch<br/>last k turns + entity memory"]
        INTENT["Intent & Slot Extraction — LLM call #1<br/>sees live df.columns + formula catalog"]
        ROUTER{"Router"}
        DQE["🟧 Dynamic Query Engine<br/>run_dynamic_query(df, spec)<br/>filter/group/agg — VECTORIZED"]
        FE["🟧 Formula Engine<br/>registry[formula_id].fn(df, params)<br/>named business formulas — VECTORIZED"]
        SEARCH["Chroma semantic search<br/>.query() metadata-filtered, or<br/>.get(ids=) direct lookup"]
        HYBRID["Hybrid / Generative<br/>combine results, or LLM drafts new copy"]
        GAP["Capability gap<br/>unknown formula / missing column<br/>→ logged, not guessed"]
        CTX["Context assembly<br/>question + calc_result + snippets + memory_summary"]
        GEN["Final Generation — LLM call #2<br/>only use numbers in calc_result;<br/>name the formula_id used"]
        RESP["Response to user"]
        UPDATE["Memory update<br/>session dict + rolling summary"]

        Q --> MEM --> INTENT --> ROUTER
        ROUTER --> DQE
        ROUTER --> FE
        ROUTER --> SEARCH
        ROUTER --> HYBRID
        ROUTER --> GAP
        DQE --> CTX
        FE --> CTX
        SEARCH --> CTX
        HYBRID --> CTX
        CTX --> GEN --> RESP --> UPDATE
        UPDATE -.next turn.-> MEM
    end

    DF -.reads.-> DQE
    DF -.reads.-> FE
    CHROMA -.reads.-> SEARCH
```

**Always applied, regardless of route:** row-level security scopes the DataFrame to the caller's portfolio *before* any filter runs · every `filter_expr` / `derived_columns.expr` is checked against an AST whitelist before it touches pandas · only `masked_mpxn` is ever embedded, never unmasked identifiers.

**🟧 = vectorized pandas operation** (whole-column ops, not row loops) · **📗 = where Excel data actually enters the system** (only here — everything downstream reads the in-memory DataFrame or Parquet cache, never the Excel file directly).
