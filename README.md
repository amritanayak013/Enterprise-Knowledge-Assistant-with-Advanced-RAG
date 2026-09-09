# Enterprise-Knowledge-Assistant-with-Advanced-RAG
An Employee Knowledge Assistant — a locally-run, production-oriented RAG (Retrieval-Augmented Generation) application that lets employees ask natural-language questions about internal policies (Leave, HR Handbook, IT, Travel, Benefits, Code of Conduct, FAQs) and get back grounded, cited answers through a conversational chat interface.


# Employee Knowledge Assistant — Enterprise RAG Application

> **GenAI Development Program — Final Evaluation Project (Project 2)**
> Advanced Retrieval-Augmented Generation (RAG) assistant for querying internal company policy documents.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Solution Overview](#2-solution-overview)
3. [Architecture Diagram](#3-architecture-diagram)
4. [Technology Stack](#4-technology-stack)
5. [Project Structure](#5-project-structure)
6. [Setup Instructions](#6-setup-instructions)
7. [Environment Variables](#7-environment-variables)
8. [How to Run the Application](#8-how-to-run-the-application)
9. [Testing (including Ollama)](#9-testing-including-ollama)
10. [Sample Inputs & Outputs](#10-sample-inputs--outputs)
11. [Key Design Decisions](#11-key-design-decisions)
12. [Limitations](#12-limitations)

---

## 1. Problem Statement

Employees waste time searching scattered PDFs/handbooks for HR/IT/Travel policy answers.
There is no single, conversational, always-available source of truth. This project
builds an assistant that answers accurately, cites its sources, and clearly says
"I don't know" instead of guessing.

## 2. Solution Overview

The **Employee Knowledge Assistant** is a locally-run RAG application that lets employees
ask natural-language questions about internal policies (Leave, HR Handbook, IT, Travel,
Benefits, Code of Conduct, FAQs) and receive grounded, cited answers through a chat interface.

**Core capabilities implemented:**
- Multi-format document ingestion (PDF + DOCX + TXT)
- Semantic chunking, embedding, and local vector storage (FAISS)
- Hybrid retrieval (vector search + BM25 keyword search)
- Reranking of retrieved candidates before final context assembly
- Conversational memory for multi-turn follow-up questions
- Source citation for every generated answer
- Streamlit chat UI with history, reset, and source display
- Hallucination guardrails ("not found in documents" fallback)

## 3. Architecture Diagram

```
Documents (PDF/DOCX/TXT)
        |
        v
  Document Loader
        |
        v
    Chunking
        |
        v
   Embeddings
        |
        v
  Vector Store (FAISS)
        |
        +------------------+
        |                  |
        v                  v
  Vector Search       BM25 Keyword Search
        |                  |
        +--------+---------+
                 v
          Hybrid Results
                 |
                 v
             Reranking
                 |
                 v
          Relevant Context
                 |
                 v
           LLM + Memory
                 |
                 v
     Grounded Answer + Sources
                 |
                 v
            Streamlit UI
```

## 4. Technology Stack

| Layer              | Technology Used                                             |
|--------------------|--------------------------------------------------------------|
| Language           | Python 3.10+                                                 |
| Document Loading   | `pdfplumber` (PDF), `python-docx` (DOCX)                      |
| Chunking           | Custom recursive character splitter                           |
| Embeddings         | `sentence-transformers` (local) or OpenAI                     |
| Vector Store       | FAISS                                                          |
| Keyword Search     | `rank-bm25`                                                    |
| Reranking          | Cross-encoder (sentence-transformers) or TF-IDF fallback       |
| LLM                | OpenAI (`gpt-4o-mini`) or Ollama (local, free)                 |
| Memory             | Custom bounded conversation buffer                             |
| UI                 | Streamlit                                                      |
| Logging            | Python `logging` module                                       |
| Version Control    | Git + GitHub                                                   |
| CI                 | GitHub Actions                                                 |

## 5. Project Structure

```
employee-knowledge-assistant/
├── data/                       # Sample source policy documents
│   ├── leave_policy.txt
│   ├── hr_handbook.docx
│   ├── travel_policy.pdf
│   └── company_faqs.txt
├── src/
│   ├── config.py                # Central .env-driven configuration
│   ├── app.py                   # Streamlit entry point
│   ├── ingestion/
│   │   ├── loaders.py
│   │   ├── chunking.py
│   │   └── build_index.py
│   ├── retrieval/
│   │   ├── embedder.py
│   │   ├── vector_store.py
│   │   ├── bm25_search.py
│   │   ├── hybrid_search.py
│   │   └── reranker.py
│   ├── generation/
│   │   ├── llm_client.py
│   │   ├── memory.py
│   │   ├── prompts.py
│   │   └── rag_pipeline.py
│   └── utils/
│       └── logger.py
├── tests/
│   ├── test_smoke.py                 # No-API sanity tests
│   └── test_ollama_connection.py     # Ollama connectivity + generation check
├── .github/workflows/ci.yml    # GitHub Actions CI pipeline
├── vector_db/                  # Persisted local vector store (gitignored)
├── requirements.txt
├── .env.example
└── README.md
```

## 6. Setup Instructions

```bash
git clone <repo-url>
cd employee-knowledge-assistant

python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate

pip install -r requirements.txt

cp .env.example .env
# Fill in required keys/values (see Section 7)
```

## 7. Environment Variables

| Variable            | Description                                   | Required |
|---------------------|------------------------------------------------|----------|
| `LLM_PROVIDER`      | `openai` or `ollama`                           | Yes      |
| `OPENAI_API_KEY`    | API key (only if LLM_PROVIDER=openai)          | Optional |
| `OLLAMA_MODEL`      | Local model name (only if LLM_PROVIDER=ollama) | Optional |
| `OLLAMA_BASE_URL`   | Ollama server URL (default localhost:11434)    | Optional |
| `EMBEDDING_PROVIDER`| `local` or `openai`                            | Yes      |
| `RERANKER_MODE`     | `cross_encoder` or `simple`                    | Yes      |
| `VECTOR_DB_PATH`    | Local path for persisted vector store          | Yes      |

**Never commit your actual `.env` file** — only `.env.example` should be in the repo.

## 8. How to Run the Application

```bash
# Step 1: Ingest documents and build the vector store
python -m src.ingestion.build_index

# Step 2: Launch the Streamlit chat interface
streamlit run src/app.py
```

## 9. Testing (including Ollama)

**Core logic smoke tests (no API/model downloads required):**
```bash
python -m pytest tests/test_smoke.py -v
```

**Ollama connectivity check** — run this before launching the app whenever
`LLM_PROVIDER=ollama`, to catch connection/model issues early with a clear message:
```bash
python tests/test_ollama_connection.py
```
This checks, in order: (1) the Ollama server is reachable, (2) the configured
model is pulled locally, (3) a real generation call succeeds. It also works as
a pytest test that skips gracefully (instead of failing) if Ollama isn't configured:
```bash
python -m pytest tests/test_ollama_connection.py -v
```

**Run the full test suite:**
```bash
python -m pytest tests/ -v
```

**CI:** `.github/workflows/ci.yml` runs the smoke tests, an Ollama integration
test (with a small model), and an ingestion build test automatically on every
push/PR.

## 10. Sample Inputs & Outputs

**Example queries to test:**
1. "What is the leave policy for new employees?"
2. "Can unused leave be carried forward to next year?" *(tests conversational memory)*
3. "What is the reimbursement limit for domestic travel?"
4. "What is the company's remote work policy?" *(tests hallucination fallback)*

```
User: What is the leave policy?
Assistant: Employees are entitled to 18 days of paid leave annually, accrued
monthly. Unused leave beyond 5 days lapses at year-end unless approved for
carry-forward by the reporting manager.

Sources:
- leave_policy.txt

User: What about carry-forward?
Assistant: Up to 5 days of unused annual leave can be carried forward to the
next calendar year with manager approval; anything beyond that lapses.

Sources:
- leave_policy.txt
```

## 11. Key Design Decisions

- **Vector store:** FAISS chosen for lightweight, dependency-minimal local persistence.
- **Hybrid search fusion:** min-max normalized weighted score combination (tunable via `HYBRID_ALPHA`), simpler to reason about and debug than reciprocal rank fusion for a small corpus.
- **Reranker:** cross-encoder for quality, with a TF-IDF fallback for fast iteration without model downloads.
- **Memory:** bounded buffer (last N turns) rather than summarization, since expected conversations are short (2-4 turns).
- **Hallucination mitigation:** strict system prompt instructing "answer only from context" plus an explicit fallback string, checked at both retrieval-empty and LLM-response levels.
- **Dual LLM support:** OpenAI and Ollama behind one interface so the project can be demoed with zero API cost.

## 12. Limitations

- Optimized for a small, curated document set; not tested at large enterprise-scale document volumes.
- Reranking adds latency vs. plain vector search; not optimized for high-concurrency use.
- No authentication or multi-user session isolation (out of scope per assessment brief).
- Conversational memory is session-based and resets when the Streamlit app restarts.
- Answer quality with Ollama/local models can be less consistent than with larger paid APIs.

---

## License / Submission Note

This project was developed as part of the **GenAI Development Program — Final Evaluation (Batch 1)**,
submitted by **Amrita Rani Nayak**, in fulfillment of Project 2 requirements.
