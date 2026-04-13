# Perencanaan Laporan UTS — II5003 Aplikasi Inteligensi Artifisial

## Agentic Document Intelligence — Expense Tracker

**Nama:** Axel David
**NIM:** 28225305
**Judul Kasus:** Agentic Document Intelligence — Expense Tracker

---

## Struktur Laporan

### 1. Pendahuluan

**Tujuan bab:** Menjelaskan konteks masalah, motivasi, dan lingkup tugas.

**Isi yang harus ditulis:**

- Latar belakang: pelacakan pengeluaran harian melibatkan beragam format dokumen (struk fisik, screenshot notifikasi bank, invoice PDF e-commerce). Proses manual lambat dan rawan kesalahan.
- Perbedaan mendasar antara:
  - **OCR biasa vs Document Intelligence** — OCR menghasilkan raw text tanpa memahami semantik, sementara Document Intelligence menambahkan layer pemahaman struktur dan konteks.
  - **Pipeline linear vs Agentic AI** — pipeline linear (OCR → regex → output) bersifat deterministik dan rigid, sementara agentic AI menggunakan reasoning, routing, dan validasi adaptif.
- Tujuan tugas: membangun sistem yang mengubah dokumen pengeluaran tidak terstruktur menjadi data terstruktur (merchant, tanggal, total) untuk otomatisasi budgeting.
- Ringkasan pendekatan: Dual comparison antara Baseline (OCR + Regex) vs Agentic (OCR + LangGraph + Ollama gemma4:e4b).

**Estimasi panjang:** 1–2 halaman

---

### 2. Dataset & Karakteristik

**Tujuan bab:** Mendeskripsikan dataset orisinal yang digunakan beserta justifikasi pemilihannya.

**Isi yang harus ditulis:**

Tabel deskripsi 7 dokumen:

| No | Nama File | Format | Sumber | Jenis | Karakteristik Unik |
|----|-----------|--------|--------|-------|---------------------|
| 1 | `doc_1_mbca.png` | PNG | Screenshot notifikasi m-BCA | Bank Notification | Teks digital, kontras tinggi, conf 91.0 |
| 2 | `doc_2_tokped.pdf` | PDF | Invoice Tokopedia | E-commerce Invoice | Dokumen bertabel, multi-kolom, conf 92.2 |
| 3 | `doc_3_email.png` | PNG | Screenshot email notifikasi bank | Bank Notification | Multi-tanggal (transaksi vs kirim), conf 79.9 |
| 4 | `doc_4_indomaret.jpg` | JPG | Foto struk Indomaret | Physical Receipt (duplikat doc_1) | Blur, noise tinggi, conf 41.2, **duplikat transaksi** |
| 5 | `doc_5_indomaret.jpeg` | JPEG | Foto struk Indomaret | Physical Receipt | Font thermal printer kecil, conf 53.6 |
| 6 | `doc_6_gacoan.jpeg` | JPEG | Foto struk ShopeeFood/Mie Gacoan | Physical Receipt | **Paling noisy**, pencahayaan buruk, conf 34.7 |
| 7 | `doc_7_shopee.pdf` | PDF | Nota pesanan Shopee | E-commerce Invoice | PDF multi-halaman, layout semi-terstruktur, conf 91.5 |

**Kriteria terpenuhi:**
- [x] Minimal 1 dokumen scan/foto blur/noise → `doc_4`, `doc_5`, `doc_6`
- [x] Minimal 1 dokumen bertabel → `doc_2`, `doc_7`
- [x] Minimal 1 dokumen semi-terstruktur (invoice) → `doc_2`, `doc_7`
- [x] Minimal 1 dokumen teks panjang → `doc_7` (multi-page)
- [x] Dataset 100% orisinal (bukan publik)

**Bukti yang harus disertakan:**
- [ ] Screenshot/gambar dari setiap dokumen asli di folder `data/raw/`
  - `[Gambar: doc_1_mbca.png — Screenshot notifikasi m-BCA]`
  - `[Gambar: doc_2_tokped.pdf — Halaman 1 invoice Tokopedia]`
  - `[Gambar: doc_3_email.png — Email notifikasi bank]`
  - `[Gambar: doc_4_indomaret.jpg — Foto struk Indomaret (blur)]`
  - `[Gambar: doc_5_indomaret.jpeg — Foto struk Indomaret]`
  - `[Gambar: doc_6_gacoan.jpeg — Foto struk Mie Gacoan (noisy)]`
  - `[Gambar: doc_7_shopee.pdf — Nota pesanan Shopee]`

**Estimasi panjang:** 2–3 halaman

---

### 3. Problem Framing

**Tujuan bab:** Mendefinisikan masalah secara jelas dan spesifik.

**Isi yang harus ditulis:**

1. **Tujuan sistem:** Mengekstrak 3 field utama (merchant, tanggal, total) dari dokumen pengeluaran multi-format secara otomatis untuk mendukung budgeting harian.

2. **Pengguna sistem:** Individu/mahasiswa yang ingin melacak pengeluaran harian dari berbagai sumber (struk belanja, notifikasi bank, invoice e-commerce) tanpa input manual.

3. **Output yang diharapkan:** JSON terstruktur per dokumen:
   ```json
   {
     "merchant": "Indomaret",
     "date": "08/04/2026",
     "total": "Rp.37.200",
     "document_type": "BANK_NOTIF",
     "is_duplicate": false
   }
   ```

4. **Tantangan utama:**
   - Variasi kualitas OCR (confidence 34.7 – 92.2)
   - Perbedaan layout antar jenis dokumen (struk fisik vs PDF digital vs screenshot)
   - Multi-entitas dalam satu dokumen (e.g., Shopee + Huawei Official Store)
   - Ambiguitas field (tanggal transaksi vs tanggal notifikasi)
   - Deteksi duplikasi antar dokumen berbeda format (doc_1 dan doc_4 = transaksi yang sama)

5. **Metrik evaluasi:**
   - Extraction Accuracy (Exact Match)
   - Field-Level Accuracy (merchant, date, total) — normalized
   - Routing Accuracy (deteksi jenis dokumen)
   - Deduplication Accuracy
   - Inference Time per dokumen

**Bukti yang harus disertakan:**
- [ ] Screenshot output problem_framing dari notebook

**Estimasi panjang:** 1–2 halaman

---

### 4. Desain Sistem

**Tujuan bab:** Menjelaskan arsitektur sistem secara keseluruhan, termasuk perbedaan pipeline baseline vs agentic.

**Isi yang harus ditulis:**

**4.1 Arsitektur Umum**

Diagram alur (buat flowchart):

```
Dokumen (PDF/JPG/PNG)
        │
        ▼
  ┌─────────────┐
  │ Preprocessing│ ← grayscale, denoise, threshold
  └──────┬──────┘
         ▼
  ┌─────────────┐
  │   OCR        │ ← pytesseract (image_to_data)
  │   (word-level│    → text, conf, position
  │    output)   │
  └──────┬──────┘
         ▼
  ┌──────────────┐
  │ Reading Order │ ← heuristic: top → left grouping
  │ Reconstruction│   → ordered_text per page
  └──────┬───────┘
         │
    ┌────┴────┐
    ▼         ▼
 Baseline   Agentic
```

**4.2 Pipeline Baseline (Non-Agentic)**

```
ordered_text → regex patterns → JSON fields
```

- Regex untuk merchant: keyword matching (Payment to, Merchant, Toko, Store)
- Regex untuk date: pattern DD/MM/YYYY, DD MMM YYYY, dsb.
- Regex untuk total: IDR/Rp + angka
- Heuristik fallback: baris pertama sebagai merchant

**4.3 Pipeline Agentic (LangGraph + Ollama)**

```
ordered_text
     │
     ▼
┌──────────────────┐
│ Orchestrator +   │ ← LLM call #1: classify + extract
│ Extractor Node   │   (gemma4:e4b via Ollama)
└────────┬─────────┘
         ▼
┌──────────────────┐
│ Validator Node   │ ← LLM call #2: format check
└────────┬─────────┘
         ▼
    JSON output
```

- LangGraph sebagai orchestrator: mengelola flow antar node
- Ollama + gemma4:e4b sebagai LLM lokal
- Dua LLM calls per dokumen (dioptimalkan dari 3 menjadi 2)
- Region classification heuristik: `key_value_line`, `amount_like`, `dense_table_like`, `paragraph_like`, `short_text`

**Bukti yang harus disertakan:**
- [ ] Diagram arsitektur (dibuat dari deskripsi di atas)
- [ ] Screenshot kode LangGraph graph definition dari notebook
- [ ] Screenshot kode region classification dari notebook

**Estimasi panjang:** 2–3 halaman

---

### 5. Implementasi

**Tujuan bab:** Menjelaskan detail teknis implementasi setiap komponen.

**Isi yang harus ditulis:**

**5.1 Environment & Dependencies**
- Python 3.10+, RAM >= 8GB
- Tesseract OCR 5.4.0
- Libraries: `pytesseract`, `pdf2image`, `Pillow`, `pandas`, `matplotlib`, `numpy`, `langchain-ollama`, `langgraph`
- LLM: gemma4:e4b via Ollama (lokal)

**5.2 Preprocessing**
- 4 strategi: `no_preprocessing`, `grayscale_only`, `standard`, `aggressive_binarize`
- Standard: grayscale → denoise → sharpening
- Aggressive binarize: threshold 180

**5.3 OCR (pytesseract)**
- `image_to_data` dengan output DICT
- Konfigurasi: OEM 3, PSM 6
- Output: `OCRRegion` dataclass (text, conf, position)
- Word-level data disimpan ke CSV

**5.4 Reading Order Reconstruction**
- Heuristic: sort by `top` → `left` per page
- `y_tolerance = 12` untuk pengelompokan baris
- Format output: `[p{page}-l{line}] text`

**5.5 Region Classification**
- Heuristik berbasis regex dan struktur teks
- 5 kategori: key_value_line, amount_like, dense_table_like, paragraph_like, short_text

**5.6 Baseline Extraction**
- Regex patterns untuk merchant, date, total
- Fallback heuristics

**5.7 Agentic Extraction**
- LangGraph graph dengan 2 node (orchestrator+extractor, validator)
- Prompt engineering untuk extraction
- Field normalization (`normalize_agentic_fields`)

**Bukti yang harus disertakan:**
- [ ] Potongan kode kunci (preprocessing, OCR, reading order, baseline regex, agentic prompt)
- [ ] Screenshot output OCR word-level data (dari `doc_7_shopee_ocr_words.csv`)
- [ ] Screenshot output reading order (ordered_text)
  - `[Gambar: Output OCR word-level — contoh doc_7_shopee]`
  - `[Gambar: Output reading order reconstruction]`

**Estimasi panjang:** 3–4 halaman

---

### 6. Eksperimen

**Tujuan bab:** Menyajikan eksperimen yang dilakukan beserta hasilnya secara sistematis.

**Isi yang harus ditulis:**

**6.1 Perbandingan Pendekatan (Baseline vs Agentic)**

Tabel hasil dari `experiment_summary.csv`:

| Dokumen | OCR Words | Avg Conf | Baseline Acc (Norm) | Agentic Acc (Norm) | Improvement |
|---------|-----------|----------|--------------------|--------------------|-------------|
| doc_1_mbca.png | 90 | 90.98 | 66.67% | 100% | +33% |
| doc_2_tokped.pdf | 182 | 92.24 | 66.67% | 100% | +33% |
| doc_3_email.png | 147 | 79.89 | 66.67% | 66% | 0% |
| doc_4_indomaret.jpg | 67 | 41.21 | 0% | 50% | +50% |
| doc_5_indomaret.jpeg | 129 | 53.60 | 50% | 100% | +50% |
| doc_6_gacoan.jpeg | 133 | 34.72 | 0% | 33% | +33% |
| doc_7_shopee.pdf | 175 | 91.53 | 66.67% | 100% | +33% |

Analisis: akurasi, fleksibilitas, error rate per pendekatan.

**6.2 Variasi Input**

Kategorisasi dokumen berdasarkan kualitas:
- **Bersih** (conf > 80): doc_1, doc_2, doc_3, doc_7 → Agentic 91.5% avg
- **Sedang** (conf 40–60): doc_4, doc_5 → Agentic 75% avg
- **Noisy** (conf < 40): doc_6 → Agentic 33%

**6.3 Variasi Strategi (Robustness Test)**

Tabel dari `robustness_results.csv`:

| Dokumen | no_preprocess (conf) | grayscale (conf) | standard (conf) | aggressive (conf) |
|---------|---------------------|-------------------|-----------------|-------------------|
| doc_1 | 91.17 | 91.17 | 90.98 | 91.60 |
| doc_2 | 92.31 | 92.31 | 92.24 | 91.98 |
| doc_3 | 81.82 | 81.82 | 79.89 | 77.90 |
| doc_4 | 41.99 | 41.99 | 41.21 | **0.00** |
| doc_5 | 58.03 | 58.03 | 53.60 | 47.93 |
| doc_6 | 40.93 | 40.93 | 34.72 | 33.65 |
| doc_7 | 91.56 | 91.56 | 91.53 | 89.97 |

Temuan kunci: aggressive binarize menghancurkan doc_4 (0 words); standard preprocessing justru menurunkan confidence pada doc_6.

**Bukti yang harus disertakan:**
- [ ] Tabel experiment_summary (dari CSV atau screenshot notebook)
- [ ] Tabel robustness_results (dari CSV atau screenshot notebook)
- [ ] `[Gambar: outputs/experiment_dashboard.png — 4-panel dashboard]`
- [ ] `[Gambar: outputs/field_accuracy_comparison.png — per-field accuracy]`
- [ ] Screenshot output processing log dari notebook (per-dokumen)

**Estimasi panjang:** 3–4 halaman

---

### 7. Evaluasi

**Tujuan bab:** Mengevaluasi hasil ekstraksi terhadap ground truth secara kuantitatif.

**Isi yang harus ditulis:**

**7.1 Ground Truth**

Tabel ground truth (dari `ground_truth.json`):

| Dokumen | Merchant | Date | Total | Type | Duplicate |
|---------|----------|------|-------|------|-----------|
| doc_1 | Indomaret | 08/04/2026 | Rp.37.200 | BANK_NOTIF | No |
| doc_2 | Tokopedia \| Qiek Store | 07/03/2026 | Rp.183.960 | ECOMMERCE | No |
| doc_3 | ESB RESTAURANT TECHNOLOGY | 06/04/2026 | Rp.60.000 | BANK_NOTIF | No |
| doc_4 | Indomaret | 08/04/2026 | Rp.37.200 | PHYSICAL_RECEIPT | **Yes** |
| doc_5 | Indomaret | 06/04/2026 | Rp.82.600 | PHYSICAL_RECEIPT | No |
| doc_6 | Mie Gacoan \| ShopeeFood | 10/04/2026 | Rp.45.500 | PHYSICAL_RECEIPT | No |
| doc_7 | Shopee \| Huawei Official Store | 05/11/2025 | Rp.335.550 | ECOMMERCE | No |

**7.2 Evaluasi Per-Field (dari `evaluation_detail.csv`)**

Tabel detail per field per dokumen dengan kolom: ground_truth, baseline_pred, baseline_exact, baseline_normalized, agentic_pred, agentic_exact, agentic_normalized.

**7.3 Ringkasan Akurasi**

| Metrik | Baseline | Agentic |
|--------|----------|---------|
| Exact Match (overall) | ~5% | ~71% |
| Normalized Match (overall) | ~45% | ~78% |
| Merchant accuracy | ~14% | ~71% |
| Date accuracy | ~43% | ~57% |
| Total accuracy | ~43% | ~86% |

**7.4 Analisis Field**

- **Merchant** paling sering salah pada baseline (regex gagal menangkap nama merchant karena variasi posisi)
- **Date** paling sulit pada agentic (hallucination: doc_3 salah tanggal, doc_6 fabrikasi total)
- **Total** paling stabil pada agentic (hanya gagal pada doc_4 dan doc_6 — keduanya low confidence)

**Bukti yang harus disertakan:**
- [ ] Tabel ground truth
- [ ] Tabel evaluation detail (dari CSV)
- [ ] `[Gambar: outputs/field_accuracy_comparison.png]`
- [ ] Screenshot output evaluasi dari notebook

**Estimasi panjang:** 2–3 halaman

---

### 8. Failure Analysis

**Tujuan bab:** Mengidentifikasi dan menganalisis kegagalan sistem secara mendalam. **Ini adalah bagian utama penilaian.**

**Isi yang harus ditulis:**

5 kegagalan yang teridentifikasi (dari `failure_analysis.json`):

**Error #1: Severe Character Misrecognition (OCR)**
- **Dokumen:** doc_6_gacoan.jpeg
- **Evidence:** Avg confidence 34.72. Baseline merchant: `\ pay i` (seharusnya ShopeeFood/Mie Gacoan). Baseline total: None.
- **Dampak:** Semua field gagal terekstrak. Baseline 0%, Agentic 33%.
- **Penyebab:** Foto struk dengan pencahayaan buruk, font thermal printer kecil, background noise tinggi. Standard preprocessing justru menurunkan confidence (40.93 → 34.72).
- **Solusi:** Adaptive preprocessing berdasarkan confidence score. Tambahkan OCR engine alternatif (EasyOCR) sebagai fallback.

**Error #2: Total OCR Failure on Binarization (OCR)**
- **Dokumen:** doc_4_indomaret.jpg
- **Evidence:** Aggressive binarize: 0 words, 0.00 confidence (total blank). Standard: 67 words, conf 41.21.
- **Dampak:** Aggressive preprocessing menghancurkan semua teks. Baseline 0%.
- **Penyebab:** Threshold binarization (180) terlalu tinggi untuk kontras rendah.
- **Solusi:** Dynamic thresholding (Otsu's method). Jangan gunakan aggressive binarize jika avg_conf < 50.

**Error #3: Line Prefix Pollution in Baseline (Reading Order)**
- **Dokumen:** doc_7_shopee.pdf
- **Evidence:** Baseline merchant: `[p1-l3] Alamat Pembeli:` — prefix tidak di-strip.
- **Dampak:** Baseline merchant 100% salah untuk semua dokumen dengan reading order format.
- **Penyebab:** `ordered_text` menggunakan format `[p1-lN] text` tapi `extract_document_fields` tidak membersihkan prefix.
- **Solusi:** Strip `[p1-lN]` prefix di awal fungsi extraction.

**Error #4: Date Hallucination (LLM Reasoning)**
- **Dokumen:** doc_3_email.png
- **Evidence:** Agentic date: 10/04/2026 (GT: 06/04/2026). LLM mengambil tanggal salah.
- **Dampak:** Agentic normalized accuracy turun ke 66.67%.
- **Penyebab:** Dokumen notifikasi bank memiliki multiple tanggal. LLM tidak membedakan tanggal transaksi vs tanggal notifikasi.
- **Solusi:** Constraint di prompt: "Pilih tanggal transaksi, bukan tanggal kirim." Few-shot examples.

**Error #5: Complete Value Fabrication (LLM Reasoning)**
- **Dokumen:** doc_6_gacoan.jpeg
- **Evidence:** Agentic date: 05/04/2023 (GT: 10/04/2026), total: Rp.500 (GT: Rp.45.500). Inference 116s.
- **Dampak:** 3/3 field salah. Data palsu berbahaya untuk budgeting.
- **Penyebab:** OCR confidence sangat rendah (34.72), LLM hallucinate dari noise.
- **Solusi:** Confidence gate: skip agentic jika conf < 40. Prompt: "Return null if unsure."

**Bukti yang harus disertakan:**
- [ ] Tabel ringkasan 5 error (id, stage, type, document, impact)
- [ ] Screenshot detail failure dari notebook
- [ ] Contoh OCR output yang salah (raw text vs expected)
- [ ] `[Gambar: Contoh OCR output doc_6_gacoan.jpeg — menunjukkan noise]`
- [ ] `[Gambar: Contoh output agentic yang hallucinate pada doc_6]`

**Estimasi panjang:** 3–4 halaman

---

### 9. Diskusi Kritis

**Tujuan bab:** Menjawab pertanyaan analisis kritis berdasarkan bukti eksperimen.

**Isi yang harus ditulis:**

**9.1 Mengapa OCR Tidak Cukup?**

Bukti:
- Baseline (OCR+Regex) normalized accuracy rata-rata ~45% vs Agentic ~78%
- doc_1: Baseline merchant `IDM INDOMARET` (noise), regex gagal normalize → Agentic: `INDOMARET` (correct)
- doc_7: Baseline merchant `Alamat Pembeli:` — regex salah tangkap baris address → Agentic: `Huawei Official Store`
- OCR menghasilkan raw text tanpa semantik. Regex terlalu rigid, tidak kontekstual, posisi-dependent.

**9.2 Kapan Agentic AI Diperlukan?**

Agentic > Baseline pada:
- doc_1 (conf 91): +33%, doc_2 (conf 92): +33%, doc_5 (conf 54): +50%, doc_7 (conf 92): +33%
- Sweet spot: dokumen multi-format dengan conf > 50

Agentic TIDAK diperlukan ketika:
- Dokumen sangat terstruktur dan template tetap
- OCR quality terlalu rendah (conf < 40): garbage-in-garbage-out

**9.3 Keterbatasan Utama Sistem**

1. **OCR Quality Gate** — korelasi kuat: conf < 50 → accuracy < 50%
2. **Inference Latency** — 49s–116s per dokumen, meningkat dengan noise
3. **LLM Hallucination** — validator hanya cek format, bukan kebenaran nilai
4. **Preprocessing Sensitivity** — doc_4 aggressive binarize: 0 words

**9.4 Apakah Sistem Scalable?**

- Throughput saat ini: ~81s/doc rata-rata (7 dokumen = 9.5 menit)
- 100 dokumen/hari: ~2.25 jam (feasible)
- 1000 dokumen/hari: ~22.5 jam (tidak feasible tanpa optimisasi)
- Bottleneck: 2 sequential LLM calls, Ollama single-request processing
- Rekomendasi: fine-tune smaller model, async batch processing, layout-aware model

**Bukti yang harus disertakan:**
- [ ] Data kuantitatif dari tabel eksperimen
- [ ] Referensi ke error spesifik dari failure analysis
- [ ] `[Gambar: outputs/experiment_dashboard.png — panel OCR conf vs accuracy]`

**Estimasi panjang:** 3–4 halaman

---

### 10. Refleksi

**Tujuan bab:** Refleksi personal berdasarkan pengalaman mengerjakan tugas.

**Isi yang harus ditulis:**

**10.1 Kesalahan yang Dilakukan**

1. Baseline extraction tidak men-strip prefix `[p1-lN]` → merchant 0% accuracy pada dokumen reading order
2. Routing `TYPE_MAP` tidak mencakup `ecommerce_invoice` → routing accuracy selalu False untuk e-commerce
3. Preprocessing dipilih statis (standard untuk semua) → padahal doc_6 justru dirugikan
4. Agentic pipeline awalnya 3 LLM calls (~5 menit/doc) → dioptimalkan menjadi 2 calls (~70s)
5. Field normalization hardcoded → menyebabkan output None sampai ditambahkan `normalize_agentic_fields`

**10.2 Insight Baru**

1. **Korelasi OCR Confidence vs Accuracy:** threshold kritis conf ~50
2. **Digital vs Foto:** gap kualitas sangat signifikan (PDF >91% conf vs foto 34–58%)
3. **Validator Node tidak efektif untuk hallucination** — hanya cek format, bukan kebenaran
4. **Preprocessing bukan selalu solusi** — agressive binarize pada doc_4 menghancurkan semua teks
5. **Merchant paling sulit** — posisi paling bervariasi antar jenis dokumen

**10.3 Perbaikan yang Diusulkan**

Prioritas tinggi:
1. Confidence gate (skip jika conf < 40)
2. Grounded validation (cek substring di OCR text)
3. Adaptive preprocessing (pilih strategy terbaik otomatis)

Prioritas sedang:
4. Few-shot prompting per document type
5. Dual OCR engine (EasyOCR fallback)
6. Template matching untuk merchant known

Prioritas rendah:
7. Fine-tune smaller model
8. Async batch processing
9. Layout-aware model (LayoutLM, Donut)

**10.4 Perbandingan Agentic vs Non-Agentic**

| Aspek | Non-Agentic (Baseline) | Agentic |
|-------|------------------------|---------|
| Accuracy (norm) | ~45% | ~78% |
| Kecepatan | Instan (<1s) | 49–116s/doc |
| Fleksibilitas | Rendah (regex rigid) | Tinggi (LLM reasoning) |
| Risiko | Gagal silent | Hallucination |
| Debugging | Transparan (regex) | Black-box (LLM) |
| Best for | Template tetap, high conf | Multi-format, conf > 50 |

**Bukti yang harus disertakan:**
- [ ] Screenshot kesalahan kode sebelum dan sesudah fix
- [ ] Data kuantitatif mendukung setiap insight
- [ ] Screenshot output refleksi dari notebook

**Estimasi panjang:** 2–3 halaman

---

## Checklist Kelengkapan Laporan

| No | Item | Status | Lokasi di Laporan |
|----|------|--------|-------------------|
| 1 | Dataset asli (7 dokumen orisinal) | ✅ | Bab 2 |
| 2 | Notebook berjalan | ✅ | `uts_agentic_expense_tracker.ipynb` |
| 3 | Ground truth dibuat | ✅ | Bab 7, `outputs/ground_truth.json` |
| 4 | Evaluasi dilakukan | ✅ | Bab 7, `outputs/evaluation_detail.csv` |
| 5 | Minimal 3 error dianalisis (ada 5) | ✅ | Bab 8, `outputs/failure_analysis.json` |
| 6 | Perbandingan agentic vs non-agentic | ✅ | Bab 6, 9, 10 |
| 7 | Refleksi ditulis | ✅ | Bab 10 |

---

## Daftar Gambar/Bukti yang Perlu Disertakan

| No | Gambar/Bukti | Sumber | Bab |
|----|-------------|--------|-----|
| 1 | Gambar 7 dokumen asli | `data/raw/` | 2 |
| 2 | Diagram arsitektur sistem | Dibuat manual/draw.io | 4 |
| 3 | Screenshot output OCR word-level | Notebook cell output | 5 |
| 4 | Screenshot reading order output | Notebook cell output | 5 |
| 5 | Experiment dashboard (4-panel) | `outputs/experiment_dashboard.png` | 6 |
| 6 | Field accuracy comparison chart | `outputs/field_accuracy_comparison.png` | 6 |
| 7 | Tabel experiment_summary | `outputs/experiment_summary.csv` | 6 |
| 8 | Tabel evaluation_detail | `outputs/evaluation_detail.csv` | 7 |
| 9 | Tabel robustness_results | `outputs/robustness_results.csv` | 6 |
| 10 | Contoh OCR noise (doc_6) | Notebook cell output | 8 |
| 11 | Contoh hallucination output (doc_6) | Notebook cell output | 8 |
| 12 | Screenshot processing log | Notebook cell output | 6 |
| 13 | Screenshot LangGraph graph def | Notebook cell | 4 |
| 14 | Screenshot output refleksi | Notebook cell output | 10 |

---

## Estimasi Total Panjang Laporan

| Bab | Estimasi |
|-----|----------|
| 1. Pendahuluan | 1–2 halaman |
| 2. Dataset & karakteristik | 2–3 halaman |
| 3. Problem framing | 1–2 halaman |
| 4. Desain sistem | 2–3 halaman |
| 5. Implementasi | 3–4 halaman |
| 6. Eksperimen | 3–4 halaman |
| 7. Evaluasi | 2–3 halaman |
| 8. Failure analysis | 3–4 halaman |
| 9. Diskusi kritis | 3–4 halaman |
| 10. Refleksi | 2–3 halaman |
| **Total** | **~22–32 halaman** |

---

## Catatan Penting

1. **Fokus pada analisis, bukan coding.** Kode ditampilkan secukupnya untuk menjelaskan pendekatan, bukan mendominasi laporan.
2. **Setiap klaim harus didukung bukti** — referensikan data dari tabel eksperimen atau output notebook.
3. **Tunjukkan error.** Justru error dan kegagalan yang dinilai, bukan kesempurnaan.
4. **Jawaban harus spesifik.** Hindari jawaban umum tanpa konteks dataset sendiri.
5. **Semua gambar yang tidak bisa disisipkan langsung** gunakan placeholder: `[Gambar: deskripsi — sumber file]`.
