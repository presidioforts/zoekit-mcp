Here’s a tight, one-page view of your two flows.

```
                  ┌──────────────────────────────────────────────────┐
                  │                 REQUEST ROUTER                   │
User Query ─────▶ │ if use_case ∈ {FAQ, ApprovedResolution} → A1     │
  (or Log Tail)   │ else → A2                                         │
                  └───────────────┬──────────────────────────────────┘
                                  │
             ┌────────────────────┴────────────────────┐
             │                                         │
             │                                         │
     A1: SME VERBATIM Q→A                      A2: DENSE RAG over LONG DOCS/LOGS
     (SentenceTransformer + Chroma)            (SentenceTransformer + Chroma)

  ┌───────────────────────────────┐          ┌───────────────────────────────────────┐
  │  Ingest (once / on update)    │          │  Ingest (once / on update)            │
  │  • Pairs: {id, query, target, │          │  • Parse docs/logs                    │
  │    version, status, sme,...}  │          │  • Chunk: ~800 tokens, 15% overlap    │
  │  • Embed TARGET only          │          │    (logs: error-anchored ±40/±80)     │
  │  • Upsert -> Chroma(collection│          │  • Metadata: kind, status, updated_at,│
  │    "faq_targets")             │          │    service/module, stage, tool, uri   │
  └───────────────┬───────────────┘          │  • Embed chunks → Chroma("kb")        │
                  │                          └─────────────────┬─────────────────────┘
                  │                                            │
  ┌───────────────▼───────────────┐          ┌─────────────────▼─────────────────────┐
  │  Query Path                    │          │  Query Path                            │
  │  1) Embed user query          │          │  1) Extract signals (tool/stage/codes) │
  │  2) Chroma top-K from         │          │  2) Build 2–3 query views:             │
  │     "faq_targets"             │          │     • original; • signal-enriched;     │
  │  3) Gate:                     │          │       • error-line only (if logs)      │
  │     top_sim≥0.50 ∧ margin≥.06 │          │  3) Dense search "kb" (K≈80 each)      │
  │     ∧ status=active           │          │  4) Union → MMR (λ≈0.5) → top 8–10     │
  └───────────────┬───────────────┘          │  5) Gate: top_sim≥0.42 ∧ margin≥.05    │
                  │                          │     ∧ has_signal (for logs)            │
        ┌─────────▼─────────┐                └───────────────┬───────────────────────┘
        │  RESPONDER        │                                │
        │  • Return TARGET  │                    ┌───────────▼───────────┐
        │    verbatim +     │                    │  RESPONDER            │
        │    provenance      │                    │  • Verbatim extract   │
        │  (no LLM rewrite) │                    │    or LLM format-only │
        └────────────────────┘                    │    (temp=0, citations)│
                                                  └───────────────────────┘
```

### Key components & settings

* **Embedding model:** `sentence-transformers/all-mpnet-base-v2` (normalize=True)
* **Vector DB:** Chroma (cosine), one row per item:

  * A1: **targets** only (`collection="faq_targets"`)
  * A2: **chunks** (`collection="kb"`)
* **Gating thresholds:** A1 `top_sim≥0.50 & margin≥0.06`; A2 `top_sim≥0.42 & margin≥0.05 (+ signal)`
* **Signals (A2 logs):** error codes, tool (maven/gradle/npm…), stage (build/test), file names, stack frames
* **MMR (A2):** λ≈0.5, keep top 8–10
* **Answer policy:**

  * A1: **VERBATIM** (include `{version, sme, approved_at, uri}`)
  * A2: **FORMAT-ONLY** or **verbatim extract**, always with **citations**

### Minimal router (pseudo)

```python
mode = "verbatim" if use_case in {"FAQ","ApprovedResolution"} else "rag_dense"
if mode == "verbatim":
    ans = retrieve_verbatim(query)   # Chroma 'faq_targets' + gate → exact target
else:
    ans = retrieve_context(query)    # Chroma 'kb', multi-view + MMR + gate
return ans
```

### Data contracts (lean)

* **A1 record:** `{id, query, target, version, status:"active", sme, approved_at, uri, hash}`
* **A2 chunk:** `{id, text, metadata:{document_id,title,uri,heading_path,kind,status,updated_at,service,module,stage,tool,hash}}`

This is ready to paste into your design doc. If you want, I can also provide a one-screen FastAPI skeleton with `/answer` routing A1 vs A2.
