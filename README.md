# L2-EdgeRAG
```mermaid
flowchart TD
  OFF["Offline: Chunking/Embedding -> IVF (centroids, posting lists) -> PQ/OPQ codebooks -> Facet Dictionary (core+dynamic). Excluded from latency metrics"] --> START

  START([Start Query]) --> QENC[Query Encoding (encoder -> q)]
  QENC --> FACET[Facet Extraction (online)\nslot-based NER + rules\nfallback: [query]]

  subgraph ACCEL[On-chip L2 Accelerator (Streaming)]
    direction TB
    FACET -->|per facet (round-robin micro-batch)| COARSE[Centroid Search (T_coarse)\ncentroids in SLM\nq_facet vs centroids -> top nprobe]
    COARSE --> POST[Posting List Load (T_candidates)\nDRAM <-> on-chip DMA\nread PQ codes/IDs (contiguous blocks)]
    POST --> APPROX[Approx Distance (ADC/LUT)\nfilter -> global M_total (in-flight)]
    APPROX --> PRECISE[Precise L2 Re-rank (T_L2)\nload exact embeds (few)\nq_facet vs candidates -> d(q,c)]
    PRECISE --> DEDUP{Early De-dup Gate\nmin_dist(c,S) < tau_dup ?}
    DEDUP -- Yes --> DROP[Drop / Strong Penalty]
    DEDUP -- No --> SCORE[score(c) = -lambda*d(q,c) + (1-lambda)*min_s d(c,s)\n(online MMR step)]
    DROP --> SEL
    SCORE --> SEL[On-chip Selector Top-K'\n(keep best K')]
    SEL --> UPDATE{Promoted?}
    UPDATE -- Yes --> SBUF[S-Buffer (Selected set)\nstore normalized embeds\n{qid, facet, valid, epoch}]
    UPDATE -- No --> NEXT[Next micro-batch or facet]
    SBUF --> EPOCH[(epoch++)\nlazy revalidate when popping]
    EPOCH --> NEXT
    NEXT -->|iterate facets| COARSE
  end

  SEL -->|end of all facets| FINAL[Finalize]
  FINAL --> SMALLMMR[Small MMR on Top-K' (optional)\nresolve ties/lazy updates]
  SMALLMMR --> TOPK[Final Top-K Chunks]
  TOPK --> CTX[Context Assembly\nbudget 600-900 tokens]
  CTX --> GEN[SLM Generate\nprefill + decode]
  GEN --> OUT([Answer])

  GEN --> LOG[Metrics Logging (per query)\nTTFT, End-to-End, T_coarse/T_candidates/T_L2/T_merge,\nP50/P95/P99, QPS, Transfer(GB), J/query,\nS1 retrieval: Recall@K, nDCG@K, Aspect-Recall,\nGen quality: EM/F1, Faithfulness, Hallucination]

