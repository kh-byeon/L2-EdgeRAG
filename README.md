# L2-EdgeRAG
```mermaid
flowchart TD
  START([Start query]) --> ENC[Encode query<br/>using embedding model]
  ENC --> FAC[Extract facets<br/>NER and rules, fallback: full query]

  subgraph ACCEL[On chip L2 accelerator<br/>Streaming rerank and MMR]
    FAC --> COARSE[Centroid search (T_coarse)<br/>find top nprobe centroids from SLM]
    COARSE --> POST[Posting list load (T_candidates)<br/>DMA transfer PQ codes and IDs]
    POST --> APPROX[Approximate distance (ADC or LUT)<br/>filter to M_total in flight]
    APPROX --> RERANK[Precise L2 rerank (T_L2)<br/>load exact embeddings and compute distance]
    RERANK --> GATE{Early de-duplication<br/>check min_dist to selected set}
    GATE -- yes --> SKIP[Skip or penalize duplicate]
    GATE -- no --> SCORE[Compute combined score<br/>relevance and diversity]
    SCORE --> SEL[Top-K Prime selector<br/>keep best K' candidates on chip]
    SEL --> PROM{Promoted to final set?}
    PROM -- yes --> SBUF[Store in S buffer<br/>normalized embedding with tags]
    PROM -- no --> NEXT[Next micro batch or facet]
    SBUF --> NEXT
    NEXT --> COARSE
  end

  SEL --> FINAL[Finalize after all facets]
  FINAL --> MMR[Small final MMR optional<br/>resolve ties or lazy updates]
  MMR --> TOPK[Final Top-K chunks<br/>ready for context assembly]
  TOPK --> CTX[Assemble context<br/>respect token budget]
  CTX --> GEN[SLM generate answer<br/>prefill and decode]
  GEN --> OUT([Return answer])
