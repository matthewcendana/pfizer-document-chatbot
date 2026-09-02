# Pharmaceutical Document Q&A

A retrieval-augmented generation pipeline that reads bundled pharmaceutical compliance PDFs, splits them back into their constituent documents, and answers plain-English questions with citations to the exact source page.

Built as the final project for **Pfizer's AI-Powered Document Insights and Data Extraction Externship** (10 weeks, 2025).

**[Watch a demo video](https://www.loom.com/share/db16d716a4784a97bf3e5bf9d9248b95)**

---

## The problem

Pharmaceutical supplier documentation arrives as a "blob" PDF: a single file that silently concatenates seven distinct document types with no index and no separators between them.

A single 10-page bundle in this project contains:

| Document                      | Pages | Contains                                                        |
| ----------------------------- | ----- | --------------------------------------------------------------- |
| Cover Letter                  | 1     | Storage and handling conditions                                 |
| Certificate of Quality (×2)   | 2–3   | Lot numbers, manufacture and expiration dates, release criteria |
| Packaging Specification       | 4–5   | Materials, part numbers, revision history                       |
| BSE/TSE Declaration           | 6     | Animal-origin compliance statement                              |
| Material Description Sheet    | 7     | Materials of construction, sterilization compatibility          |
| Supplier Qualification Record | 8–9   | Audit history, ISO certifications, approved products            |
| Chain of Custody              | 10    | Manufacturing site, distribution flow                           |

To answer one compliance question — _"is this material free of animal origin, and what is it made of?"_ — a quality engineer has to know which two documents to open, then read both. And in a regulated environment the answer alone isn't enough: it has to be traceable to a specific page for audit.

Some of these arrive as scans with no searchable text at all.

---

## What this does

```
Document Input → OCR → Chunking → Embeddings → Vector Storage → Retrieval → LLM Response
```

**Ingestion — runs once per document.** A PDF is checked for a usable text layer; pages
without one go through OCR. Both paths converge on segmentation, chunking, embedding, and
storage in a global index plus one index per document type.

<p align="center">
  <img src="docs/ingestion.jpeg" alt="Ingestion pipeline" width="700">
</p>

**Query — runs per question.** The LLM predicts which document type holds the answer. Above
0.7 confidence it searches that type's index alone; below, it falls back to searching
everything. The top four chunks are labelled with their source before the LLM sees them.

<p align="center">
  <img src="docs/query.jpeg" alt="Query pipeline" width="700">
</p>

1. **Splits the bundle.** An LLM classifies each page into one of seven document types and detects where one document ends and the next begins. No templates, no hardcoded page ranges.
2. **Handles scans.** Any page yielding fewer than 40 characters of extractable text is rendered at 200 DPI and passed to Tesseract OCR, so scanned and digital documents enter the same pipeline.
3. **Routes queries.** Before searching, the LLM predicts which document type is likely to hold the answer. Above 0.7 confidence it searches only that document type's index; below, it falls back to searching everything.
4. **Cites its sources.** Retrieved passages carry `[From DocType, Pages X–Y]` labels the model is instructed to copy verbatim, making every citation both human-readable and automatically verifiable.
5. **Refuses when it should.** When the retrieved context doesn't contain the answer, the model is instructed to say so rather than guess.

Everything runs locally. No API key, no external account, no document text leaving the machine — which is what makes it deployable against supplier documentation under NDA.

---

## Stack

| Component        | Choice                              | Configuration                                          |
| ---------------- | ----------------------------------- | ------------------------------------------------------ |
| **OCR**          | Tesseract 5 (pytesseract) + PyMuPDF | Triggers below 40 chars of text layer; 200 DPI         |
| **Chunking**     | Custom sliding window               | 220 words, 60-word overlap; never spans sub-documents  |
| **Embeddings**   | all-MiniLM-L6-v2                    | 384-dim, L2-normalised so inner product = cosine       |
| **Vector store** | FAISS `IndexFlatIP`                 | Exact search; one global index + one per document type |
| **Retriever**    | Dense search + LLM query routing    | top-k = 4; routes above 0.7 confidence, else global    |
| **LLM**          | Qwen2.5-3B-Instruct (local)         | fp16, greedy decoding for reproducibility              |
| **UI**           | Gradio                              | File upload, document breakdown, chat with sources     |

---

## Results

Evaluated on 4 documents (2 digital, 2 scanned), 26 pages, 17 questions, 34 total question runs.

| Metric                                    | Digital | Scanned | Overall  |
| ----------------------------------------- | ------- | ------- | -------- |
| Answer accuracy                           | 77%     | 85%     | **81%**  |
| Correct refusal on unanswerable questions | 100%    | 100%    | **100%** |
| Citation accuracy (right source cited)    | 100%    | 100%    | **100%** |
| Citation validity (no fabricated sources) | 100%    | 100%    | **100%** |
| Retrieval Recall@3                        | 93%     | 89%     | **91%**  |
| Retrieval Hit Rate@3                      | 100%    | 100%    | **100%** |
| Mean Reciprocal Rank                      | 0.94    | 0.94    | **0.94** |
| Document-type classification recall       | 100%    | 100%    | **100%** |
| Avg response time                         | 4.4s    | 4.9s    | **4.7s** |
| Avg LLM generation                        | 3.4s    | 3.9s    | **3.7s** |

### The interesting finding

**Every error was over-refusal, not invention.** Four of the five failures retrieved the correct document and then replied _"the provided documents do not contain enough information."_ The information was in the context window.

So the refusal instruction that produced 100% on the trick questions is also causing false refusals on answerable ones. The system's failure mode is excessive caution rather than hallucination — which, for a regulated use case, is the failure mode you'd choose if you had to pick one. But it's a real cost, and it's the top item on the roadmap below.

Full per-question results, timings and generated answers: [`results/results.md`](results/results.md).

---

## Repository

```
├── notebooks/
│   ├── 01_pipeline_and_demo.ipynb   # The pipeline + Gradio UI — start here
│   └── 02_evaluation.ipynb          # Metrics, ground truth, ablation
├── data/
│   ├── pharma-blob-sample.pdf       # 10-page bundle, all 7 document types
│   ├── sample-sdf-document.pdf      # 3-page bundle
│   └── ground_truth.json            # Answer key: 17 questions
├── results/
│   ├── results.md                   # Full evaluation output
│   └── metrics.json                 # Headline numbers
└── docs/
    ├── ingestion.jpeg               # Architecture: document ingestion path
    └── query.jpeg                   # Architecture: query and answer path
```

---

## Running it

**In Colab** (recommended — GPU, no local setup):

1. Open `notebooks/01_pipeline_and_demo.ipynb` in Colab
2. **Runtime → Change runtime type → T4 GPU**
3. **Runtime → Run all.** Model download takes ~3 minutes
4. Open the Gradio link from the last cell, upload a PDF from `data/`, click **Process Document**

It runs on CPU without a GPU, just slowly (roughly 20–60s per answer). Switching `LLM_MODEL_NAME` to `Qwen/Qwen2.5-1.5B-Instruct` roughly halves that.

**To reproduce the evaluation:** upload both PDFs and `ground_truth.json` to `/content`, then run `02_evaluation.ipynb`. Takes 20–35 minutes on a T4. The scanned test copies are generated automatically.

### Try these questions

- _"What is the lot number and expiration date for the AKTA ready Low Flow Kit?"_
  A straightforward lookup — but there are **two** certificates in the file, so the retriever has to pick the right one. Returns lot 18356721, expiration 20260315, cited to page 2.

- _"What is the lot number and expiration date for part number 29477427?"_
  Part 29477427 is a real product referenced across five of the seven documents, but its Certificate of Quality is **missing from the bundle**. The system should refuse rather than borrow a lot number from a neighbouring certificate. In a real compliance workflow, "this bundle is incomplete" is the useful signal.

---

## Limitations

These are real, measured, and worth stating plainly.

**The evaluation is small.** 4 documents, 26 pages, ~19 indexed chunks. Retrieving 3 passages covers roughly a quarter of the corpus, so **Recall@3 of 91% and Hit Rate of 100% are inflated by corpus size** and would not survive a larger document population. Hit Rate and MRR carry more signal here than Recall@K. Precision@3 of 52.8% looks low but is a ceiling artifact: most questions have only 1–2 relevant chunks in the whole file, so retrieving 3 caps precision mathematically.

**The scanned documents are synthetic.** They're rasterised from the digital originals rather than genuine archival scans. Tesseract recovers essentially all of the text from them, so the digital-vs-scanned comparison is a weaker test than real scans with stamps, handwriting and heavy skew would be. The upside: since OCR isn't losing text, any gap between the columns points at classification, chunking or retrieval rather than extraction.

**Query routing selects exactly one document type.** Questions spanning two documents can only ever retrieve from half the relevant source. Observed directly: a multi-hop question about materials _and_ animal origin retrieved only the Material Description Sheet, then answered the animal-origin half anyway and attributed it to a document that doesn't contain that claim. Every automatic check passed. A reviewer opening that page would not find the statement.

**The faithfulness metric grades itself.** Factual consistency is judged by the same model that wrote the answer — a lenient grader, not an independent audit. It missed the misattribution described above. Treat 65% as a smoke detector, not a measurement.

**Classification errors cap everything downstream.** In earlier runs the Chain of Custody document was misclassified as a second Certificate of Quality, so that document type never entered the index and every question about manufacturing site or distribution flow became unanswerable regardless of retrieval quality. Boundary detection compares consecutive pages with no global view of the document.

**Not production infrastructure.** In-memory FAISS with no persistence, single-user, re-processes every document per session, no authentication, no audit logging. This is a working prototype for a 10-week externship, not a deployable system.

---

## Roadmap

**Short-term**

- Keyword prior on classification — document titles here are highly regular, so regex on the first 200 characters with the LLM as tiebreaker should eliminate misclassification cheaply
- Route to the **top two** document types and merge results, targeting the cross-document gap directly. The ablation harness already exists to measure whether it helps
- Soften the refusal threshold to reduce false refusals without reintroducing hallucination — measure both together, since they trade off

**Medium-term**

- Cross-encoder reranking over the top 10 chunks; measure the precision gain against added latency
- Replace the self-judge with an independent grader model or human-adjudicated answers
- Expand to genuine scanned documents and a larger corpus until Recall@K stops saturating
- Calibrate routing confidence against observed hit rate rather than setting the threshold by intuition

**Longer-term**

- Persistent vector store with incremental indexing, so a corpus is built once and added to
- Structured field extraction alongside free-text Q&A — lot numbers, expiration dates and certification validity as typed fields, enabling automated checks like _"flag every supplier whose ISO certification has lapsed."_ (In this corpus, both of Cytiva Sweden AB's ISO certifications show a validity date of 2024-12-31.)

---

## Notes

Sample documents are publicly available Cytiva product documentation, used here for demonstration. The pipeline is document-agnostic — the seven document types are defined in `VALID_DOC_TYPES` and the classification prompt, both editable in one place.

**Requires Gradio 6.** The `Chatbot` component no longer accepts a `type` argument, and `theme` belongs on `launch()` rather than `Blocks()`.
