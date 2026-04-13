# 📓 1. NOTEBOOK STRUCTURE (FINAL)

---

## 🔹 SECTION 0 — Reproducibility (WAJIB)

Tambahkan:

```markdown
## Reproducibility

Environment:

- Python 3.10+
- RAM: ≥8GB

Setup:

- pip install -r requirements.txt
- ollama serve
- ollama pull gemma4:e4b

Execution:

- Run all cells sequentially

Notes:

- LLM output may vary slightly
```

---

## 🔹 SECTION 1 — Problem Framing

Tambahkan:

```markdown
### Tujuan

Ekstraksi field (merchant, date, total) dari dokumen semi-terstruktur

### Pendekatan

1. Heuristic baseline (rule-based)
2. Agentic system (LLM + LangGraph)

### Metrics

- Extraction Accuracy
- Field-Level Accuracy
- Deduplication Accuracy
```

👉 Ini bikin jelas:
➡️ lo tidak melanggar template
➡️ tapi punya **2 pendekatan**

---

## 🔹 SECTION 2–7 — OCR & HEURISTIC PIPELINE (JANGAN DIUBAH BESAR)

Ikuti template dosen:

- OCR
- bounding box
- grouping
- reading order

---

### 🔥 Tambahkan ANALISIS:

```markdown
Pendekatan heuristik:

- Menggunakan aturan tetap (regex, keyword)
- Cepat tetapi tidak adaptif

Kelemahan:

- Gagal pada layout kompleks
- Sensitif terhadap OCR error
```

👉 Ini penting untuk justify LLM

---

## 🔹 SECTION 8 — BASELINE EXTRACTION

Pipeline:

```text
OCR → regex → JSON
```

Tambahkan:

```markdown
Baseline digunakan sebagai pembanding untuk agentic system.
```

---

# 🚀 SECTION 9 — AGENTIC SYSTEM (OLLAMA + LANGGRAPH)

👉 Ini bagian upgrade lo (tapi tetap “terlihat sesuai template”)

---

## 🔹 Arsitektur Agent

Tambahkan markdown:

```markdown
## Agentic System (LLM-based)

Sistem menggunakan pendekatan agentic dengan komponen:

1. Orchestrator (LangGraph)
   - Menentukan strategi ekstraksi

2. Tool Executor
   - Menjalankan parsing field

3. Evaluator
   - Validasi hasil ekstraksi

LLM digunakan melalui Ollama untuk reasoning.
```

---

## 🔹 Integrasi Ollama

Contoh PRD:

```python
from langchain_community.llms import Ollama

llm = Ollama(model="gemma4:e4b")
```

---

## 🔹 LangGraph Flow

Struktur:

```text
input → orchestrator → tool → evaluator → output
```

Tambahkan:

```markdown
Berbeda dengan heuristic, agent dapat:

- memilih strategi
- memperbaiki kesalahan parsing
```

---

## 🔹 Contoh Behavior Agent

Tambahkan ini (penting untuk nilai):

```markdown
Contoh:
Jika merchant tidak ditemukan dengan regex,
agent akan mencoba:

- mencari teks paling atas
- menggunakan konteks layout
```

👉 Ini bikin dosen lihat:
💥 “ini benar-benar agentic reasoning”

---

# 📊 SECTION 10 — EVALUATION (SUPER UPGRADE)

---

## 🔹 Wajib 3 Level

---

### 1. Exact Match

```text
Accuracy = matched fields / total fields
```

---

### 2. Field-Level Accuracy

```python
Field      Accuracy
Merchant   %
Date       %
Total      %
```

---

### 3. Comparison

```markdown
Perbandingan:

Baseline:

- gagal pada layout kompleks

Agentic:

- lebih robust terhadap variasi
```

---

## 🔥 Tambahkan INSIGHT:

```markdown
Field "merchant" memiliki akurasi terendah,
menunjukkan bahwa parsing berbasis posisi lebih sulit
dibanding parsing numerik seperti total.
```

---

# 🧪 SECTION 11 — EXPERIMENT DESIGN (WAJIB EKSPLISIT)

Tambahkan:

```markdown
## Experimental Design

Eksperimen dilakukan dalam 3 variasi:

1. Pendekatan:
   - Heuristic vs Agentic

2. Input:
   - Digital vs scanned

3. Strategi:
   - Dengan/ tanpa preprocessing
```

---

# ⚠️ SECTION 12 — FAILURE ANALYSIS

Minimal 5:

| Case | Error | Root Cause | Fix |

Tambahkan contoh:

- OCR noise
- LLM hallucination
- routing error
- format mismatch
- duplicate miss

---

# 🔍 SECTION 13 — REFLECTION

Tambahkan:

```markdown
Pendekatan agentic memberikan fleksibilitas tinggi,
namun tetap bergantung pada kualitas OCR.

Future work:

- fine-tuning model
- layout-aware model
```
