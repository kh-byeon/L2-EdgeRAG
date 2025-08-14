# L2-EdgeRAG
```mermaid
flowchart TD
  %% === Inputs / Offline ===
  OFF[("Offline\n• Chunking/Embedding\n• IVF (centroids, posting lists)\n• PQ/OPQ codebooks\n• Facet Dictionary (core+dynamic)\n— Excluded from latency metrics —"):::off] --- START

  START([Start Query]) --> QENC[Query Encoding\n(encoder → q)]
  QENC --> FACET[Facet Extraction (online)\nslot-based NER + rules\nif empty → fallback = [query]]

  subgraph ACCEL["On‑chip L2 Accelerator (Streaming)"]
    direction TB
    FACET -->|for each facet (round‑robin micro‑batch)| COARSE[Centroid Search (T_coarse)\ncentroids in SLM\nq_facet vs centroids → top nprobe]
    COARSE --> POST[Posting List Load (T_candidates)\nDRAM ←→ on‑chip DMA\nread PQ codes/IDs (contiguous blocks)]
    POST --> APPROX[Approx Distance (ADC/LUT)\nfilter → global M_total (in‑flight)]
    APPROX --> PRECISE[Precise L2 Re‑rank (T_L2)\nload exact embeds (few)\nq_facet vs candidates → d(q,c)]
    PRECISE --> DEDUP{Early De‑dup Gate\nmin_dist(c,S) < τ_dup ?}
    DEDUP -- yes --> DROP[Drop / Strong Penalty]
    DEDUP -- no --> SCORE[Score(c) = -λ·d(q,c) + (1-λ)·min_s∈S d(c,s)\n(online MMR step)]
    DROP --> SEL
    SCORE --> SEL[On‑chip Selector Top‑K′\n(tournament/heap)\nkeep only best K′]
    SEL --> UPDATE{Promoted?}
    UPDATE -- yes --> SBUF[S‑Buffer (Selected set)\nstore normalized embeds\n{qid, facet, valid, epoch}]
    UPDATE -- no --> NEXT[Next micro‑batch or facet]
    SBUF --> EPOCH[(epoch++)\nlazy revalidate when popping]
    EPOCH --> NEXT
    NEXT -->|iterate facets| COARSE
  end

  SEL -->|end of all facets| FINAL[Finalize]
  FINAL --> SMALLMMR[Small MMR on Top‑K′ (optional)\n(ties/lazy updates resolved)]
  SMALLMMR --> TOPK[Final Top‑K Chunks]
  TOPK --> CTX[Context Assembly\nparent‑child expand, budget 600–900 tokens]
  CTX --> GEN[SLM Generate\nprefill + decode]
  GEN --> OUT([Answer])

  %% Logging / Metrics
  classDef off fill:#efefef,stroke:#bbb,color:#444;
  classDef log fill:#fff8e1,stroke:#f1b700,color:#444;

  GEN --> LOG[Metrics Logging (per query)\n• TTFT, End‑to‑End\n• T_coarse / T_candidates / T_L2 / T_merge\n• P50/P95/P99, QPS\n• Transfer(GB), J/query\n• S1 retrieval quality: Recall@K, nDCG@K, Aspect‑Recall\n• Gen quality: EM/F1, Faithfulness, Hallucination]:::log
