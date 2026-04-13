# Agentic Omnichannel Expense Tracker

**UTS II5003 Aplikasi Inteligensi Artifisial — Institut Teknologi Bandung**

Axel David — 28225305

---

## Overview

An Agentic Document Intelligence system that converts unstructured expense documents (physical receipts, bank notifications, e-commerce invoices) into structured JSON data for automated daily budgeting. The system compares two extraction approaches side-by-side:

| Approach | Pipeline | Avg Accuracy |
|----------|----------|:------------:|
| **Baseline (Non-Agentic)** | OCR &rarr; Regex &rarr; JSON | 42.9% |
| **Agentic** | OCR &rarr; LLM Reasoning &rarr; LangGraph Orchestration &rarr; JSON | 71.4% |

The agentic approach achieves **+28.6% improvement** over baseline across 7 real-world documents with varying quality (OCR confidence 34–92).

## System Architecture

```
Document (PDF/JPG/PNG)
        │
        ▼
┌────────────────┐
│ Preprocessing  │  4 strategies: no_preprocess, grayscale, standard, aggressive_binarize
└───────┬────────┘
        ▼
┌────────────────┐
│ OCR            │  pytesseract (image_to_data) → word-level: text, confidence, position
└───────┬────────┘
        ▼
┌────────────────┐
│ Reading Order  │  Heuristic: top → left grouping, y_tolerance=12
│ Reconstruction │
└───────┬────────┘
        │
   ┌────┴─────┐
   ▼          ▼
Baseline    Agentic
(regex)     (LangGraph + Ollama gemma4:e4b)
   │          │
   │     ┌────┴────────────┐
   │     │ Orchestrator +  │  LLM call #1: classify + extract
   │     │ Extractor Node  │
   │     └────┬────────────┘
   │          ▼
   │     ┌────────────┐
   │     │ Validator   │  LLM call #2: format validation
   │     │ Node        │
   │     └────┬────────┘
   │          │
   ▼          ▼
  JSON      JSON
```

## Dataset

7 original documents (no public datasets) covering 3 categories:

| # | Document | Format | Type | OCR Confidence |
|---|----------|--------|------|:--------------:|
| 1 | `doc_1_mbca.png` | PNG | Bank Notification (m-BCA) | 90.98 |
| 2 | `doc_2_tokped.pdf` | PDF | E-commerce Invoice (Tokopedia) | 92.24 |
| 3 | `doc_3_email.png` | PNG | Bank Notification (email) | 79.89 |
| 4 | `doc_4_indomaret.jpg` | JPG | Physical Receipt (blur, duplicate of #1) | 41.21 |
| 5 | `doc_5_indomaret.jpeg` | JPEG | Physical Receipt | 53.60 |
| 6 | `doc_6_gacoan.jpeg` | JPEG | Physical Receipt (noisy, low light) | 34.72 |
| 7 | `doc_7_shopee.pdf` | PDF | E-commerce Invoice (Shopee) | 91.53 |

## Key Findings

### Accuracy by Document Quality

| Quality Tier | OCR Confidence | Agentic Accuracy | Baseline Accuracy |
|:------------:|:--------------:|:----------------:|:-----------------:|
| Clean | > 80 | ~91% | ~67% |
| Medium | 40–60 | ~75% | ~25% |
| Noisy | < 40 | ~33% | 0% |

### Field-Level Accuracy

| Field | Baseline | Agentic |
|-------|:--------:|:-------:|
| Merchant | 29% | 100% |
| Date | 57% | 43% |
| Total | 43% | 71% |

### Failure Analysis (5 identified errors)

| # | Stage | Error Type | Document |
|---|-------|------------|----------|
| 1 | OCR | Severe Character Misrecognition | doc_6_gacoan.jpeg |
| 2 | OCR | Total Failure on Binarization | doc_4_indomaret.jpg |
| 3 | Reading Order | Line Prefix Pollution | doc_7_shopee.pdf |
| 4 | LLM Reasoning | Date Hallucination | doc_3_email.png |
| 5 | LLM Reasoning | Complete Value Fabrication | doc_6_gacoan.jpeg |

## Project Structure

```
.
├── 28225305_UTS_Aplikasi AI.ipynb   # Main notebook (run all cells sequentially)
├── data/
│   └── raw/                         # Original documents (not tracked, private)
├── outputs/
│   ├── doc_*_full_result.json       # Per-document extraction results (7 files)
│   ├── doc_7_shopee_*.csv/json      # Active document detailed outputs
│   ├── experiment_summary.csv       # All documents overview
│   ├── evaluation_detail.csv        # Per-field evaluation (exact + normalized)
│   ├── ground_truth.json            # Manual ground truth
│   ├── failure_analysis.json        # 5 failure cases with evidence
│   ├── robustness_results.csv       # 4 preprocessing strategies × 7 documents
│   ├── experiment_dashboard.png     # 4-panel summary dashboard
│   └── field_accuracy_comparison.png
├── LAPORAN_PLAN.md                  # Report planning document
└── README.md
```

## Requirements

- Python 3.10+
- RAM >= 8GB
- [Tesseract OCR](https://github.com/UB-Mannheim/tesseract/wiki) installed locally
- [Ollama](https://ollama.com/) running locally

### Python Dependencies

```
pytesseract
pdf2image
Pillow
pandas
matplotlib
numpy
langchain-ollama
langgraph
```

## Quick Start

```bash
# 1. Install Tesseract OCR (Windows)
# Download from https://github.com/UB-Mannheim/tesseract/wiki

# 2. Install Ollama and pull the model
ollama serve
ollama pull gemma4:e4b

# 3. Install Python dependencies
pip install pytesseract pdf2image pillow pandas matplotlib numpy langchain-ollama langgraph

# 4. Place your documents in data/raw/

# 5. Run the notebook
jupyter notebook "28225305_UTS_Aplikasi AI.ipynb"
```

## License

Academic use only — UTS II5003 ITB.
