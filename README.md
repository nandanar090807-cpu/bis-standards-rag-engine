# bis-standards-rag-engine
# BIS Standards Recommendation Engine

> AI-powered RAG pipeline that maps plain-English product descriptions to relevant Bureau of Indian Standards (IS) codes in under 100ms — built for the **BIS × Sigma Squad AI Hackathon 2026**.

**GitHub:** https://github.com/nandanar090807-cpu/bis-standards-rag-engine

---

## Evaluation Results

Evaluated on 50 queries (10 public + 40 synthetic across cement, steel, concrete, aggregate, brick, pipe categories).

| Metric | Score | Target | Status |
|---|---|---|---|
| Hit Rate @3 | 88.00% | > 80% | ✅ PASS |
| MRR @5 | 0.8180 | > 0.70 | ✅ PASS |
| Avg Latency | 0.030s | < 5.0s | ✅ PASS |
| Test queries | 50 | — | — |

Public test set (10 queries): **Hit@3 100%, MRR@5 1.000**.

**Latency note:** The 0.030s average is measured without query expansion (no `GEMINI_API_KEY`). With a key set, each query includes a Gemini round-trip (~1–3s extra). Both modes satisfy the <5s target. The first query after startup runs slower (~0.15s) due to PyTorch JIT compilation; subsequent queries run at steady-state (~0.027s).

---

## Quickstart (for judges)

```bash
pip install -r requirements.txt
```

### ⚠️ Step 1 — Build the index (required before inference)

Place the BIS SP 21 PDF (from the hackathon rulebook) in the project root as `dataset.pdf`, then run:

```bash
python3 src/ingest.py --pdf dataset.pdf
```

This parses all 929 pages, extracts 564 IS standard chunks, embeds them with `all-MiniLM-L6-v2`, and stores them in a local ChromaDB vector store. Takes ~2–3 minutes on first run.

### Step 2 — Run inference

```bash
python3 inference.py --input hidden_private_dataset.json --output team_results.json
```

This is the **judge entry point**. Replace `hidden_private_dataset.json` with the private test set path.

### Step 3 — Evaluate

```bash
python3 eval_script.py --results team_results.json
```

To reproduce results on the included 50-query eval set:

```bash
python3 eval_script.py --results data/eval_output.json
```

---

## Architecture

**Ingestion** (`src/ingest.py`): BIS SP 21 (929-page PDF) is parsed with PyMuPDF using `SUMMARY OF` section headers as chunk boundaries, yielding one chunk per IS standard (564 total). Each chunk is embedded with `all-MiniLM-L6-v2` and stored in ChromaDB with cosine similarity indexing.

**Retrieval** (`src/retriever.py`, `inference.py`): BM25Okapi and bi-encoder vector search run in parallel. Scores are normalised and blended (85% vector, 15% BM25; `alpha=0.15`). A `keyword_rescore` pass then blends the hybrid score with TF-weighted keyword overlap (alpha=0.3). A `family_rerank` pass promotes the correct Part within IS-number families using frequency-weighted keyword overlap. Results are deduplicated and the top-5 IS codes returned.

**Demo UI** (`app.py`): Gradio interface with optional Gemini 2.5 Flash rationale generation.

---

## Retrieval Strategy

Pure semantic search struggles with closely related standards (e.g. IS 269 vs IS 8112 vs IS 12269 — all OPC cement variants). BM25 provides exact keyword signal, and `family_rerank` handles Part disambiguation within the same IS number family. Together these steps recover roughly 15 percentage points of Hit@3 over pure semantic retrieval.

---

---

## Project Structure
## Project Structure

```text
bis_rag_submission/
├── inference.py         # Judge entry point
├── eval_script.py      # Evaluation metrics (Hit@3, MRR@5, latency)
├── app.py              # Gradio demo UI
├── requirements.txt    # Dependency list
├── run_eval.sh         # One-command evaluation helper
├── src/
│   ├── config.py       # Hyperparameters and paths
│   ├── ingest.py       # PDF parsing + ChromaDB ingestion
│   ├── retriever.py    # Hybrid BM25 + vector retrieval
│   ├── generator.py    # Gemini rationale + query expansion
│   └── utils.py        # Shared utilities
├── scripts/
│   └── generate_eval.py # Synthetic test set generator
└── data/
    ├── public_test_set.json   # 10 public evaluation queries
    ├── eval_ready.json        # 50-query evaluation set (10 public + 40 synthetic)
    └── eval_output.json       # Pre-run evaluation results

---

## Environment Variables

Copy .env.example to .env then add your key:

GEMINI_API_KEY=your_key_here

Inference and evaluation run fully offline without GEMINI_API_KEY.

---

## Tech Stack

| Component | Library |
|---|---|
| PDF Parsing | PyMuPDF (fitz) |
| Embeddings | all-MiniLM-L6-v2 (SentenceTransformers) |
| Vector Store | ChromaDB cosine similarity |
| Keyword Retrieval | BM25Okapi (rank-bm25) |
| LLM (optional) | Gemini 2.5 Flash via REST API |
| Demo UI | Gradio |

---

Nandan A R — Student, IIT Madras | BIS x Sigma Squad AI Hackathon 2026
