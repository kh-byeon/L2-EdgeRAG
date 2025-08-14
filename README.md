# L2-EdgeRAG
```mermaid
flowchart TD
  OFF["Offline: Chunking and Embedding -> IVF (centroids, posting lists) -> PQ or OPQ codebooks -> Facet dictionary. Excluded from latency metrics"] --> START

  START([Start Query]) --> QENC[Query Encoding]
  QENC --> FACET[Facet Extraction (online)\nslot based NER and rules\nfallback: use full query]

  subgraph ACCEL[On chip L2 Accelerator (Streaming)]
    direction TB
    FACET -->|per facet round robin micro batch| COARSE[Centroid Search T_coarse\ncentroids in SLM\nfacet query vs centroids to top nprobe]
    COARSE --> POST[Posting List Load T_candidates\nDRAM <-> on chip DMA\nread PQ codes and IDs]
    POST --> APPROX[Approx Distance ADC or LUT\nfilter to global M_total in flight]
    APPROX --> PRECISE[Precise L2 Re rank T_L2\nload exact embeddings few\nfacet query vs candidates to d(q,c)]
    PRECISE --> DEDUP{Early De dup Gate\nmin_dist(c,S) < tau_dup ?}
    DEDUP -- Yes --> DROP[Drop or Strong Penalty]
    DEDUP -- No --> SCORE[score(c) = -lambda * d(q,c) + (1 - lambda) * min_s d(c,s)\nonline MMR step]
    DROP --> SEL
    SCORE --> SEL[On chip Selector TopKPrime\nkeep best KPrime]
    SEL --> UPDATE{Promoted}
    UPDATE -- Yes --> SBUF[S Buffer Selected set\nstore normalized embeddings\nqid, facet, valid, epoch]
    UPDATE -- No --> NEXT[Next micro batch or facet]
    SBUF --> EPOCH[(epoch plus plus)\nlazy revalidate when popping]
    EPOCH --> NEXT
    NEXT --> COARSE
  end

  SEL --> FINAL[Finalize]
  FINAL --> SMALLMMR[Small MMR on TopKPrime optional\nresolve ties and lazy updates]
  SMALLMMR --> TOPK[Final Top K Chunks]
  TOPK --> CTX[Context Assembly\nbudget 600 to 900 tokens]
  CTX --> GEN[SLM Generate\nprefill and decode]
  GEN --> OUT([Answer])

  GEN --> LOG[Metrics Logging per query\nTTFT, End to End, T_coarse, T_candidates, T_L2, T_merge,\nP50 P95 P99, QPS, Transfer GB, J per query,\nS1 retrieval: Recall at K, nDCG at K, Aspect Recall,\nGeneration: EM F1 Faithfulness Hallucination]

