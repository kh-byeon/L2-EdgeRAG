# L2-EdgeRAG
```mermaid
flowchart TD
  START([Start]) --> ENC[Encode query]
  ENC --> FAC[Extract facets]

  subgraph ACCEL[On chip L2 accelerator]
    FAC --> COARSE[Centroid search]
    COARSE --> POST[Load posting lists]
    POST --> APPROX[Approx distance]
    APPROX --> RERANK[Precise L2 rerank]
    RERANK --> GATE{Early dedup}
    GATE -- yes --> SKIP[Skip]
    GATE -- no --> SCORE[Compute score]
    SCORE --> SEL[TopKPrime selector]
    SEL --> PROM{Promoted}
    PROM -- yes --> SBUF[S buffer]
    PROM -- no --> NEXT[Next batch or facet]
    SBUF --> NEXT
    NEXT --> COARSE
  end

  SEL --> FINAL[Finalize]
  FINAL --> MMR[Small MMR optional]
  MMR --> TOPK[Final Top K]
  TOPK --> CTX[Assemble context]
  CTX --> GEN[Generate]
  GEN --> OUT([Answer])
