# CatalogueIQ Evaluation Report

## 1. Dataset Summary

- 500 product records across 5 categories.
- Categories: Electronics, Fashion, Home & Kitchen, Beauty, and Books.
- Source formats: CSV, Markdown, HTML.
- Product review summaries: CSV.
- Raw chunks: 633.
- Fixed-size chunks: 1184.
- Structure-aware chunks: 633.
- Approximate total tokens: 38,395.
- Embedding model: `text-embedding-3-small`.
- Vector store: FAISS `IndexFlatIP`.

## 2. Chunking Decision

Two chunking strategies were implemented and inspected. Fixed-size chunking produced 1184 chunks and can split related product fields across chunk boundaries.

Structure-aware chunking produced 633 chunks. Each catalogue row is kept as one semantic product chunk, while Markdown and HTML content is split around meaningful document boundaries.

Structure-aware chunking was selected because it preserves relationships between product attributes such as price, battery life, warranty, seller information, and return window.

## 3. Retrieval Design

- Query expansion generates multiple variants for every query.
- E-commerce synonyms include earphones, earbuds, TWS, and in-ear headphones.
- Metadata filtering supports category, price range, product type, and battery threshold.
- SQLite provides exact filtering for structured product constraints.
- FAISS provides dense vector retrieval.
- BM25 provides lexical retrieval for exact names and identifiers.
- Reciprocal-rank fusion combines dense and lexical retrieval.
- Improved retrieval depth: top-k=8.

## 4. Context Formatting and Persona Prompts

Retrieved chunks use `[SRC: source | section_or_product_id]` citation markers. The generation prompt instructs the model to answer only from retrieved context, avoid invented facts, cite factual claims, and state when the available evidence is insufficient.

### Shopper

```text
You are CatalogueIQ for shoppers. Answer only from the supplied ShopSmart context. Compare products using only retrieved facts. Never invent specifications, warranty terms, sellers, stock, or policy rules. Cite factual claims using the supplied [SRC: ...] markers. If the context is insufficient, say so clearly.
```

### Seller

```text
You are CatalogueIQ for ShopSmart sellers. Answer only from the supplied ShopSmart context. Focus on listing, image, pricing, category, compliance, and seller-policy requirements. Never invent requirements. Cite factual claims using [SRC: ...].
```

### Support Agent

```text
You are CatalogueIQ for internal support agents. Answer only from the supplied ShopSmart context. For multi-hop questions, connect relevant policy sections explicitly. Do not create exceptions not present in the context. Cite factual claims using [SRC: ...].
```

## 5. Baseline vs Improvement

- Baseline retrieval hit rate: 0.800
- Improved retrieval hit rate: 0.850
- Delta: +0.050

The baseline used dense FAISS retrieval with top-k=3 and no query expansion. The improved pipeline added query expansion, BM25/FAISS hybrid retrieval, deeper retrieval, metadata filtering, and SQLite-backed numeric constraints.

## 6. RAGAS Evaluation

RAGAS was executed across all 20 evaluation questions using faithfulness, answer relevancy, and context precision.

### Overall Scores

| Metric | Baseline | Improved | Delta |
|---|---:|---:|---:|
| Faithfulness | 0.868 | 0.853 | -0.015 |
| Answer Relevancy | 0.837 | 0.928 | +0.091 |
| Context Precision | 0.825 | 0.839 | +0.014 |

### Results by Query Type

| Query Type                 |   Baseline Faithfulness |   Baseline Answer Relevancy |   Baseline Context Precision |   Improved Faithfulness |   Improved Answer Relevancy |   Improved Context Precision |   Faithfulness Delta |   Answer Relevancy Delta |   Context Precision Delta |
|:---------------------------|------------------------:|----------------------------:|-----------------------------:|------------------------:|----------------------------:|-----------------------------:|---------------------:|-------------------------:|--------------------------:|
| Comparative recommendation |                   0.833 |                       0.656 |                        0.208 |                   0.876 |                       0.906 |                        0.302 |                0.043 |                    0.250 |                     0.094 |
| Multi-hop reasoning        |                   0.713 |                       0.671 |                        0.958 |                   0.610 |                       0.880 |                        0.965 |               -0.103 |                    0.209 |                     0.006 |
| Policy & eligibility       |                   1.000 |                       0.970 |                        0.958 |                   0.938 |                       0.970 |                        0.927 |               -0.062 |                   -0.000 |                    -0.031 |
| Product factual lookup     |                   0.875 |                       0.958 |                        1.000 |                   0.875 |                       0.958 |                        1.000 |                0.000 |                    0.000 |                    -0.000 |
| Seller policy lookup       |                   0.917 |                       0.929 |                        1.000 |                   0.964 |                       0.926 |                        1.000 |                0.048 |                   -0.003 |                     0.000 |

### Interpretation

The improved retrieval pipeline increased retrieval hit rate from 0.800 to 0.850.

Answer relevancy changed from 0.837 to 0.928, while context precision changed from 0.825 to 0.839.

Faithfulness changed from 0.868 to 0.853. This indicates a trade-off: the improved retrieval pipeline provided a broader and more relevant candidate set, but the generation stage did not consistently use the additional context as faithfully as the baseline.

This result is treated as an engineering finding rather than claiming that every evaluation metric improved simultaneously.

## 7. What Did Not Work

1. Fixed-size chunking can separate related product facts across chunks.
2. Very small retrieval depth is insufficient for comparative recommendation queries.
3. Dense-only retrieval is less effective for some exact product names and identifiers.

## 8. Lessons Learned

- Structured and unstructured sources require different chunking boundaries.
- Comparative recommendation quality depends strongly on candidate-set recall.
- Structured numeric constraints are better handled with database filtering.
- Hybrid retrieval improves robustness for exact product matching.
- More retrieved context does not automatically produce more faithful answers.

## 9. Scalability Considerations

With 100,000 additional products, the main pressure points would be embedding generation time, vector-index size, and retrieval latency.

Batched embedding generation, persistent caching, incremental FAISS updates, and database-backed filtering would help maintain performance.

## 10. Conclusion

CatalogueIQ demonstrates a RAG-based product intelligence assistant over mixed structured and unstructured ShopSmart India data. The system supports product factual lookup, policy and eligibility questions, comparative recommendations, seller policy lookup, and multi-hop reasoning while providing grounded answers and source citations.