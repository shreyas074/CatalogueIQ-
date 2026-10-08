# CatalogueIQ — RAG-Powered Product Intelligence Assistant

**ShopSmart India | Capstone Level 2**

CatalogueIQ is a Retrieval-Augmented Generation (RAG) product intelligence assistant built for the fictional **ShopSmart India** e-commerce platform.

It combines product catalogue data with returns and refunds policies, promotional terms, seller onboarding guidance, category documentation, buyer FAQs, and product review summaries. The system retrieves relevant evidence and uses an LLM to generate grounded answers with source citations.

---

## Features

* Product factual lookup
* Product comparison and recommendations
* Returns and refund policy questions
* Seller policy and onboarding questions
* Multi-hop reasoning across catalogue and policy information
* Out-of-catalogue / insufficient-evidence handling
* Three user personas:

  * Shopper
  * Seller
  * Support Agent
* Conversation preference memory
* Metadata-based filtering
* Hybrid dense + lexical retrieval
* Query expansion
* Source citations
* RAGAS evaluation
* Gradio-based user interface

---

## Architecture

```text
                    User Query
                        │
                        ▼
              ┌──────────────────┐
              │ Persona Selection│
              │ Shopper/Seller/  │
              │ Support Agent    │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │ Query Processing │
              │ & Expansion     │
              └────────┬─────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       FAISS          BM25       SQLite
    Dense Search   Lexical Search Filters
          │            │            │
          └────────────┼────────────┘
                       ▼
              Reciprocal Rank Fusion
                       │
                       ▼
               Retrieved Context
                       │
                       ▼
              Grounded LLM Prompt
                       │
                       ▼
                Generated Answer
                       │
                       ▼
              Source Citations
```

---

## Knowledge Base

The system creates a deterministic fictional ShopSmart India dataset containing:

* **500 products**
* Electronics
* Fashion
* Home & Kitchen
* Beauty
* Books
* Product review summaries
* Returns and refunds policy
* Big Billion Days promotional terms
* Seller onboarding guide
* Category taxonomy
* Category-specific attribute guides
* Buyer FAQ with 50 questions

The product catalogue contains structured fields such as:

* Product ID
* Product name
* Brand
* Category
* Product type
* Price
* Rating
* Battery life
* ANC support
* Warranty
* Return window
* Seller ID
* Seller verification
* Product tags

The dataset is generated with a fixed random seed so that the same input data is produced on every run.

---

## Data Sources

CatalogueIQ demonstrates ingestion from multiple formats:

```text
CSV
 ├── Product catalogue
 └── Product review summaries

Markdown
 ├── Returns & refunds policy
 ├── Promotional terms
 ├── Seller onboarding guide
 ├── Category taxonomy
 └── Category attribute guides

HTML
 └── Buyer FAQ
```

Each source is converted into a common `DocumentChunk` representation containing:

```text
text
source
section
doc_type
metadata
```

Metadata is later used for filtering and source attribution.

---

## Chunking Strategy

Two chunking approaches were implemented and compared.

### 1. Fixed-size Chunking

Documents are split into fixed character windows with overlap.

```text
Document
   ↓
180-character chunks
   ↓
30-character overlap
```

This approach is simple but can separate related product attributes or policy information.

### 2. Structure-aware Chunking

The final system uses structure-aware chunking.

* Product catalogue rows remain intact.
* Review records remain intact.
* FAQ sections remain intact.
* Markdown documents are split around meaningful document boundaries.
* Long text is divided at sentence boundaries.

This preserves semantic relationships between related fields such as:

```text
Price
Rating
Battery
Warranty
Seller
Return window
```

Structure-aware chunking was therefore selected for the final retrieval pipeline.

---

## Embeddings

CatalogueIQ uses OpenAI embeddings:

```text
Model: text-embedding-3-small
```

Embeddings are normalized before indexing so that inner-product similarity can be used as cosine similarity.

### Embedding Cache

Embeddings are cached using a content hash.

```text
Document chunks
      ↓
Content hash
      ↓
Check cache
   ↙       ↘
Exists    Missing
  ↓          ↓
Load      OpenAI API
cache         ↓
  │        Save cache
  └───────┬───────┘
          ▼
       FAISS
```

This prevents unchanged documents from being unnecessarily embedded again.

---

## Retrieval System

CatalogueIQ uses multiple retrieval mechanisms.

### FAISS

FAISS performs dense semantic retrieval using the OpenAI embeddings.

It is useful for questions where the wording differs from the wording used in the knowledge base.

### BM25

BM25 provides lexical retrieval.

It is particularly useful for:

* Exact product names
* Product IDs
* Brand names
* Specific terminology
* Keyword-heavy questions

### SQLite

SQLite is used for structured catalogue filtering.

It handles constraints such as:

```text
Category
Price range
Rating
Brand
Product type
Battery threshold
```

This is especially useful for comparison and recommendation queries.

---

## Hybrid Retrieval

The improved retrieval pipeline combines dense and lexical retrieval.

```text
                    Query
                      │
              Query Expansion
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
       FAISS                    BM25
    Dense Search           Lexical Search
          │                       │
          └───────────┬───────────┘
                      ▼
             Reciprocal Rank
                  Fusion
                      │
                      ▼
              Top-k Results
```

The improved pipeline retrieves up to **8 chunks**, compared with **3 chunks** in the baseline.

---

## Query Expansion

Before retrieval, the system generates alternative interpretations of the user's query.

For e-commerce terminology, the system understands related terms such as:

```text
earphones
earbuds
TWS
in-ear headphones
```

This increases the probability of retrieving relevant catalogue records even when the user uses different terminology from the source documents.

---

## Metadata Filtering

Catalogue metadata allows the system to apply structured constraints.

For example:

```text
"Wireless earbuds under ₹3000 with ANC and
at least 20 hours battery life"
```

can be translated into constraints such as:

```text
Product type = Wireless Earbuds
Price <= ₹3000
ANC = Yes
Battery >= 20 hours
```

SQLite narrows the structured product candidates while semantic and lexical retrieval provide supporting evidence.

---

## Grounded Generation

Retrieved chunks are formatted with source information before being sent to the LLM.

Example:

```text
[SRC: product_catalogue.csv | P1002]

Product ID: P1002
Name: Sony WF-C700N
Brand: Sony
Category: Electronics
Price: ₹2799
Rating: 4.5/5
Battery life: 30 hours
ANC: Yes
Warranty: 12 months
```

The generation prompt instructs the model to:

* Answer using retrieved evidence.
* Avoid inventing facts.
* Cite factual claims.
* State when the available evidence is insufficient.

---

## Out-of-Catalogue Handling

CatalogueIQ does not blindly answer questions about products or facts that cannot be verified.

A similarity threshold is used to determine whether sufficient evidence exists.

If relevant evidence is not found, the system responds with an explicit insufficiency message rather than hallucinating product specifications or policy rules.

---

## Personas

CatalogueIQ provides three personas.

### Shopper

Designed for:

* Product searches
* Product comparisons
* Recommendations
* Product specifications
* Returns and refunds

### Seller

Designed for:

* Seller onboarding
* Listing requirements
* Category requirements
* Product image requirements
* Seller policies

### Support Agent

Designed for:

* Customer support
* Policy interpretation
* Returns
* Refunds
* Buyer protection
* Multi-step customer issues

Each persona uses a different prompt style while sharing the same retrieval and knowledge base.

---

## Conversation Memory

The shopper flow includes lightweight conversation memory.

The system can track preferences such as a preferred brand and incorporate that preference into subsequent product searches.

For example:

```text
User:
I prefer Sony products.

Later:

User:
Which earbuds should I buy?
```

The system can incorporate the stored preference into the effective retrieval query.

---

## Recommendation Flow

Recommendation queries are handled using the structured product index.

Example:

```text
Which wireless earbuds under ₹3000
have ANC and at least 20 hours battery?
```

The system:

```text
User Query
    ↓
Identify recommendation intent
    ↓
Apply structured catalogue constraints
    ↓
Filter eligible products
    ↓
Rank candidates
    ↓
Retrieve supporting evidence
    ↓
Generate grounded response
```

This allows numerical constraints to be handled more reliably than relying entirely on semantic retrieval.

---

## Gradio Interface

The project provides a simple Gradio interface.

```text
CatalogueIQ — ShopSmart India

Persona:
[ Shopper ▼ ]

Ask CatalogueIQ:
[........................................]

[ Ask ]

Answer:
...........................................

Retrieved Evidence:
...........................................
```

The evidence panel is intentionally exposed so that users and evaluators can inspect the information retrieved by the RAG pipeline.

---

## Evaluation

The evaluation dataset contains **20 questions**, balanced across five query types:

| Query Type                 | Questions |
| -------------------------- | --------: |
| Product factual lookup     |         4 |
| Policy & eligibility       |         4 |
| Comparative recommendation |         4 |
| Seller policy lookup       |         4 |
| Multi-hop reasoning        |         4 |
| **Total**                  |    **20** |

The system evaluates both baseline and improved retrieval.

### Baseline

```text
FAISS
Top-k = 3
No query expansion
No hybrid fusion
```

### Improved

```text
Query expansion
+
FAISS
+
BM25
+
Reciprocal-rank fusion
+
SQLite filtering
+
Top-k = 8
```

The notebook measures retrieval hit rate before running the external RAGAS evaluation.

---

## RAGAS Evaluation

CatalogueIQ evaluates the generated answers using three RAGAS metrics:

### Faithfulness

Measures whether the generated answer is supported by the retrieved context.

### Answer Relevancy

Measures whether the response appropriately addresses the user's question.

### Context Precision

Measures how relevant the retrieved context is to the question.

Both baseline and improved retrieval pipelines are evaluated.

---

## Technology Stack

| Component              | Technology                      |
| ---------------------- | ------------------------------- |
| Language               | Python                          |
| LLM                    | OpenAI                          |
| Embeddings             | OpenAI `text-embedding-3-small` |
| Vector Search          | FAISS                           |
| Lexical Search         | BM25                            |
| Structured Filtering   | SQLite                          |
| RAG Evaluation         | RAGAS                           |
| UI                     | Gradio                          |
| Data Processing        | Pandas, NumPy                   |
| HTML Parsing           | BeautifulSoup                   |
| Environment Management | python-dotenv                   |

---

## Project Structure

```text
CatalogueIQ/
│
├── CatalogueIQ_Final_Submission.ipynb
├── evaluation_report.md
├── README.md
├── requirements.txt
│
├── shopsmart_data/
│   ├── product_catalogue.csv
│   ├── product_review_summaries.csv
│   ├── returns_refunds_policy.md
│   ├── big_billion_days_terms.md
│   ├── seller_onboarding_guide.md
│   ├── category_taxonomy.md
│   ├── *_attributes.md
│   └── buyer_faq.html
│
└── shopsmart_cache/
    ├── embeddings_*.npy
    └── products.db
```

---

## Installation

Create and activate a Python virtual environment:

```bash
python -m venv venv
```

### Windows

```bash
venv\Scripts\activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

---

## API Key Configuration

Create a `.env` file:

```env
OPENAI_API_KEY=your_openai_api_key
```

Optionally specify the generation model:

```env
CATALOGUEIQ_OPENAI_MODEL=gpt-4o-mini
```

The notebook uses the OpenAI API for:

* Text embeddings
* LLM generation
* RAGAS evaluation

---

## Running the Project

Open the notebook:

```text
CatalogueIQ_Final_Submission.ipynb
```

Run the cells from top to bottom.

The notebook will:

```text
1. Install dependencies
2. Load environment variables
3. Generate ShopSmart data
4. Load CSV/Markdown/HTML sources
5. Create chunks
6. Compare chunking strategies
7. Generate embeddings
8. Build the FAISS index
9. Build the SQLite product index
10. Build BM25 retrieval
11. Configure query expansion
12. Configure hybrid retrieval
13. Configure personas and memory
14. Launch the Gradio interface
15. Run retrieval evaluation
16. Run RAGAS evaluation
17. Generate evaluation_report.md
18. Generate README.md
```

---

## Example Queries

### Product Lookup

```text
What is the battery life of the boAt Airdopes 141?
```

### Product Comparison

```text
Which wireless earbuds under ₹3000 have ANC and at least
20 hours battery life?
```

### Policy

```text
Can I return a fashion item bought on sale during
Big Billion Days?
```

### Seller

```text
What images are required for an Electronics listing?
```

### Multi-hop

```text
If I buy a smartwatch from a third-party seller and it
stops working after 45 days, what are my options?
```

### Out-of-catalogue

```text
What is the battery life of a product that is not present
in the ShopSmart catalogue?
```

The system should indicate when the available knowledge base cannot verify the requested information.

---

## Key Design Decisions

### Why RAG?

Product and policy information changes frequently. RAG allows the assistant to retrieve relevant information from the current ShopSmart knowledge base rather than relying entirely on information stored inside the language model.

### Why structure-aware chunking?

Product records naturally form semantic units. Keeping product attributes together improves retrieval for questions involving multiple constraints.

### Why hybrid retrieval?

Dense retrieval is good at semantic similarity, while BM25 is effective for exact terms such as product names and IDs. Combining both improves retrieval robustness.

### Why SQLite?

Numerical constraints such as price, rating, and battery life are better handled with structured queries than semantic similarity alone.

### Why source citations?

Citations make the generated response auditable and allow users to inspect the evidence used by the RAG system.

---

## Limitations

* The ShopSmart catalogue is synthetic and generated for the capstone.
* Product information is not connected to a live e-commerce database.
* OpenAI API access is required for embedding and LLM generation.
* RAGAS evaluation requires an available LLM/embedding API configuration.
* The Gradio interface is intended as a demonstration rather than a production deployment.
* Conversation memory is lightweight and session-oriented rather than a persistent user profile system.

---

## Conclusion

CatalogueIQ demonstrates an end-to-end RAG architecture for e-commerce product intelligence.

The system combines:

```text
Multi-format ingestion
        ↓
Structure-aware chunking
        ↓
OpenAI embeddings
        ↓
FAISS semantic retrieval
        +
BM25 lexical retrieval
        +
SQLite structured filtering
        ↓
Hybrid evidence retrieval
        ↓
Persona-aware prompting
        ↓
Grounded LLM generation
        ↓
Source-cited answers
        ↓
RAGAS evaluation
```

The project demonstrates how a product catalogue can be transformed into a searchable and grounded AI assistant capable of answering factual, policy, seller, comparative, and multi-hop questions.
