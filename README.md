# L2-EdgeRAG
```mermaid
flowchart TD
  OFF["Offline: Chunking + Embedding; IVF centroids and posting lists; PQ or OPQ codebooks; Facet dictionary. Excluded from latency metrics"] --> START

  START([Start Query]) --> QENC[Query Encoding]
  QENC --> FACET[Facet Extraction online<br/>slot based NER and rules<br/>fallback: use full query]

  subgraph ACCEL[On chip L2 Accelerator Streaming]
    direction TB
    FACET -->|per facet round robin micro batch| COARSE[Centroid Search T_coarse<br/>centroids in SLM<br/>facet query vs centroids to top nprobe]
    COARSE --> POST[Posting List Load T_candidates<br/>DRAM and on chip DMA<br/>read PQ codes and IDs]
    POST --> APPROX[Approx Distance ADC or LUT<br/>filter to global M_total in flight]
    APPROX --> PRECISE[Precise L2 Re rank T_L2<br/>load exact embeddings few<br/>facet query vs candidates to distance]
    PRECISE --> DEDUP{Early De dup Gate<br/>min_dist c,S less than tau_dup ?}
    DEDUP -- Yes --> DROP[Drop or Strong Penalty]
    DEDUP -- No --> SCORE[score c = -lambda * d q,c + (1 - lambda) * min_s d c,s<br/>online MMR step]
    DROP --> SEL
    SCORE --> SEL[On chip Selector TopKPrime<br/>keep best KPrime]
    SEL --> UPDATE{Promoted}
    UPDATE -- Yes --> SBUF[S Buffer Selected set<br/>store normalized embeddings<br/>qid facet valid epoch]
    UPDATE -- No --> NEXT[Next micro batch or facet]
    SBUF --> EPOCH[(epoch plus plus)<br/>lazy revalidate when popping]
    EPOCH --> NEXT
    NEXT --> COARSE
  end

  SEL --> FINAL[Finalize]
  FINAL --> SMALLMMR[Small MMR on TopKPrime optional<br/>resolve ties and lazy updates]
  SMALLMMR --> TOPK[Final Top K Chunks]
  TOPK --> CTX[Context Assembly<br/>budget 600 to 900 tokens]
  CTX --> GEN[SLM Generate<br/>prefill and decode]
  GEN --> OUT([Answer])

  GEN --> LOG[Metrics Logging per query<br/>TTFT End to End T_coarse T_candidates T_L2 T_merge<br/>P50 P95 P99 QPS Transfer GB J per query<br/>S1 retrieval Recall at K nDCG at K Aspect Recall<br/>Generation EM F1 Faithfulness Hallucination]
