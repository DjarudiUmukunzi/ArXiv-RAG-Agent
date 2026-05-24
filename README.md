### ArXiv AI Research RAG Pipeline

> An end-to-end Retrieval-Augmented Generation (RAG) system that ingests recent AI research papers from ArXiv, indexes them semantically, and answers natural language queries with cited, grounded responses.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Pipeline Stages](#pipeline-stages)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Setup](#setup)
- [Configuration](#configuration)
- [Usage](#usage)
- [Streamlit UI](#streamlit-ui)
- [Evaluation](#evaluation)
- [Models](#models)
- [License](#license)

---

## Overview

Veridocs fetches papers from the ArXiv `cs.AI` category, chunks and embeds them, stores them in a ChromaDB vector database, and uses the **Anthropic Claude API** to generate answers backed by retrieved sources. A Streamlit interface makes the system fully accessible without writing any code.

Each answer is classified as **Supported**, **Partially Supported**, or **Not Supported** based on retrieval quality and citation coverage, giving users a transparent confidence signal alongside every response.

---

## Architecture

```
ArXiv API
    │
    ▼
Data Ingestion & Chunking        ← feedparser, NLTK, semantic chunking
    │
    ▼
Embedding Generation              ← sentence-transformers/all-MiniLM-L6-v2
    │
    ▼
Vector Store (ChromaDB)           ← persistent cosine-similarity index
    │
    ▼
Retrieval (Top-K)                 ← query → embedding → nearest chunks
    │
    ▼
LLM Generation                   ← Claude Haiku via Anthropic API
    │
    ▼
Answer + Citations + Confidence
    │
    ▼
Streamlit UI  ──────────────────  Evaluation Framework
```

---

## Pipeline Stages

| Stage | File | Description |
|-------|------|-------------|
| 1–2 | `data_ingestion.py` | Fetch papers from ArXiv API; semantic chunking with sentence boundaries |
| 3–4 | `embeddings.py` | Batch embedding with `all-MiniLM-L6-v2`; persist to ChromaDB |
| 5–6 | `rag_core.py` | Cosine-similarity retrieval; LLM answer generation with inline citations |
| 7–8 | *(evaluation)* | Auto-generated benchmark Q&A; confidence scoring; failure analysis |
| UI  | `app.py` | Streamlit chat interface with analytics dashboard |

---

## Project Structure

```
FinalProject/
├── config.py                        # Centralised configuration
├── data_ingestion.py                # Stages 1-2: ArXiv fetch + chunking
├── embeddings.py                    # Stages 3-4: Embeddings + ChromaDB indexing
├── rag_core.py                      # Stages 5-6: RAG agent (retrieval + generation)
├── main.py                          # Orchestration entry point
├── app.py                           # Streamlit UI
├── requirements.txt                 # Python dependencies
├── stage1.ipynb – stage5.ipynb      # Exploratory notebooks (per stage)
├── Sprint2.md                       # Sprint progress log
├── Rag_lab.md                       # Lab notes
└── arxiv_cs_ai/
    ├── papers_metadata.jsonl        # Raw paper metadata
    ├── chunks.jsonl                 # Semantic text chunks
    ├── chunk_embeddings.npy         # Embedding vectors
    ├── chunks_metadata.parquet      # Chunk metadata (parquet format)
    ├── chroma_db/                   # Persistent ChromaDB vector store
    ├── benchmark.csv                # Auto-generated evaluation questions
    └── evaluation_results.csv       # Evaluation output
```

---

## Requirements

| Requirement | Detail |
|-------------|--------|
| Python | 3.10+ |
| OS | macOS / Linux |
| RAM | 8 GB minimum |
| Internet | Required for ArXiv fetch and Anthropic API calls |
| Anthropic API Key | Required — set as `ANTHROPIC_API_KEY` environment variable |

---

## Setup

### 1. Clone the repository

```bash
git clone <your-repo-url>
cd FinalProject
```

### 2. Create and activate a virtual environment

```bash
python -m venv venv
source venv/bin/activate        # macOS / Linux
# venv\Scripts\activate         # Windows
```

### 3. Set your Anthropic API key

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```

Add this to your shell profile (`.zshrc`, `.bashrc`, etc.) to persist it across sessions.

### 4. Install dependencies

```bash
pip install -r requirements.txt
```

Or install manually:

```bash
pip install feedparser pandas numpy sentence-transformers chromadb \
            anthropic pyarrow nltk tenacity \
            scikit-learn streamlit seaborn
```

---

## Configuration

All settings live in `config.py`. Key options:

| Setting | Default | Description |
|---------|---------|-------------|
| `CATEGORY` | `cs.AI` | ArXiv category to fetch |
| `MAX_RESULTS` | `200` | Number of papers to fetch |
| `CHUNK_SIZE` | `150` | Token size per chunk |
| `CHUNK_OVERLAP` | `30` | Overlap between consecutive chunks |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Sentence transformer for embeddings |
| `LLM_MODEL` | `claude-haiku-4-5-20251001` | Claude model used for answer generation |
| `TOP_K` | `5` | Retrieved chunks per query |
| `MAX_NEW_TOKENS` | `512` | Maximum generated tokens per answer |
| `TEMPERATURE` | `0.7` | Sampling temperature for LLM |
| `BENCHMARK_SAMPLES` | `20` | Questions used in evaluation |

---

## Usage

### Run the full pipeline end-to-end

```bash
python main.py --step all
```

### Run individual stages

```bash
python main.py --step ingest       # Fetch and chunk ArXiv papers
python main.py --step embeddings   # Generate embeddings and index into ChromaDB
python main.py --step rag          # Run a RAG demo with sample queries
python main.py --step eval         # Run the evaluation benchmark
python main.py --step ui           # Launch the Streamlit UI
```

---

## Streamlit UI

Launch the interactive chat interface:

```bash
streamlit run app.py
```

Then open [http://localhost:8501](http://localhost:8501) in your browser.

### UI Features

- **Chat tab** — Submit natural language questions; view answers with inline citations, confidence scores, response times, and expandable source cards.
- **Analytics tab** — Session-level charts for response time trends and answer quality distribution; one-click CSV export.
- **About tab** — System overview, architecture summary, and usage tips.
- **Sidebar controls** — Adjust Top-K, temperature, max output tokens, and a recency filter (last 30 days) without touching code.

---

## Evaluation

Veridocs ships with a built-in evaluation framework that:

- Auto-generates 20 benchmark question-answer pairs from indexed papers
- Runs all benchmark questions through the full RAG pipeline
- Computes per-answer metrics:
  - **Support label** — `Supported` / `Partially Supported` / `Not Supported`
  - **Confidence score** — weighted combination of retrieval similarity, answer length, and citation count
  - **Citation count** — number of inline paper references in the answer
- Produces summary plots (label distribution, correctness rate) via matplotlib/seaborn
- Writes results to `arxiv_cs_ai/evaluation_results.csv`

---

## Models

| Component | Model | Notes |
|-----------|-------|-------|
| Embeddings | `sentence-transformers/all-MiniLM-L6-v2` | Runs locally; CPU-friendly |
| LLM | `claude-haiku-4-5-20251001` | Anthropic API — fast, cost-efficient |

The embedding model runs **locally** via `sentence-transformers`. The LLM is called **remotely** through the Anthropic API — no GPU required. To switch Claude models, update `LLM_MODEL` (and `MODEL_ID`) in `config.py`.

---

# References

## [1] Anthropic — Claude (AI Coding Assistant)

Claude by Anthropic was used throughout this project to assist with code development,
debugging, refactoring, and implementation of the RAG pipeline components
(data ingestion, embeddings, retrieval, evaluation, and Streamlit UI).

> Anthropic. (2024). *Claude AI Assistant*. Anthropic, PBC.  
> https://www.anthropic.com/claude

---

## [2] IBM — Building AI Agents

IBM's Think educational resources on AI agent design patterns were used as a
learning reference for understanding how to architect the RAG agent, including
the retrieve-then-generate pattern, confidence scoring, and answer verification.

> IBM. (2024). *What are AI agents?* IBM Think.  
> https://www.ibm.com/think/topics/ai-agents

> IBM. (2024). *What is retrieval-augmented generation?* IBM Think.  
> https://www.ibm.com/think/topics/retrieval-augmented-generation

---

## [3] Hugging Face — Models, Transformers & Sentence-Transformers

Hugging Face provided the pre-trained models and open-source libraries central
to this project:
---

## License

MIT License
