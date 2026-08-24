# RAG + Calculation-Backend Chatbot — Flow

Excel/pandas as the structured store · ChromaDB as the vector store · no SQL anywhere

## How to read this

- Excel loads once into pandas (numbers) and ChromaDB (text) — top row.
- Each question goes through the router to the right engine, then the results get combined into one answer — bottom row.
- Dashed arrows = engines reading already-loaded data, never touching Excel directly.
- 🟧 = vectorized pandas step · 📗 = where Excel enters the system

```mermaid
flowchart TB

    EXCEL["📗 Excel file"] --> PANDAS["🟧 pandas DataFrame<br/>(numbers, in memory)"]
    EXCEL --> CHROMA["ChromaDB<br/>(text, embedded)"]

    Q["User question"] --> INTENT["Intent extraction<br/>(LLM)"]
    INTENT --> ROUTER{"Router"}

    ROUTER --> DQE["🟧 Query engine<br/>filter/group/aggregate"]
    ROUTER --> FE["🟧 Formula engine<br/>named calculations"]
    ROUTER --> SEARCH["Chroma search<br/>(explanations)"]

    PANDAS -.-> DQE
    PANDAS -.-> FE
    CHROMA -.-> SEARCH

    DQE --> COMBINE["Combine results"]
    FE --> COMBINE
    SEARCH --> COMBINE

    COMBINE --> GEN["Final answer<br/>(LLM, numbers only)"]
    GEN --> RESP["Response to user"]
```

**Always applied:** results are filtered to the caller's own portfolio first, every query is checked before it touches pandas, and only masked customer IDs ever get embedded.

## When formula needs query's output first

Sometimes it's not query *or* formula — it's query *then* formula, on the same rows (e.g. "efficiency index for my top 20 by propensity"). The router then runs them as a chain instead of two separate branches:

```mermaid
flowchart LR
    ROUTER{"Router"} --> DQE["🟧 Query engine<br/>e.g. top 20 by score"]
    DQE --> FE["🟧 Formula engine<br/>runs only on those 20 rows"]
    FE --> COMBINE["Combine results"]
```
