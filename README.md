# LLM Bias Audit Pipeline

Script to **collect**, **process**, and **analyze** ranked LLM recommendations from OpenAI, Google Gemini, and Anthropic, then merge them with benchmark/price metadata and run regression + diagnostics to test for **self-preferential bias**.

---

## Start

1. **Clone** into a project folder.  
2. **Create a `.env`** with your API keys (if collecting data).  
3. Decide what you want to run and **comment/uncomment the sections inside `main()`**.  
4. For the full regression on existing CSVs, leave **Section 6** enabled and run:
   ```bash
   python3 main.py
   ```
5. Outputs will land under various `figures_*`, and `output_datasets_coeffs/`.

---

## What this script does

- **Section 1 (Pilot collection)** — Optional, paid: query OpenAI/Gemini/Claude with a single prompt per category across many iterations, save raw top-3 model rankings.  
- **Section 2 (Pilot variation plots)** — Optional: read the pilot runs and plot **variation** (how stable the top-3 is as iterations increase).  
- **Section 3 (Main data collection)** — Optional, paid: bulk collection across many prompts & categories; writes vendor CSVs in `datasets/`.  
- **Section 4 (Prelim figures)** — Optional: quick heatmaps / CI bars from the `datasets/` CSVs.  
- **Section 5 (Processing)** — Required for analysis: normalize/model-alias mapping, join with benchmark + cost metadata, and **write expanded CSVs**.  
- **Section 6 (Analysis)** — Required for analysis: OLS + ordered-logit, VIF, correlations, PCA loadings, permutation test for `isSelfPromoted`, per-category runs, and **save coefficient tables & figures**.

You can turn sections on/off by commenting blocks inside `main()`.

---

## Installation
**Dependencies:**
```bash
python-dotenv
openai
google-generativeai
anthropic
pandas
numpy
matplotlib
seaborn
scikit-learn
statsmodels
plotly
```

Install:
```bash
pip install -r requirements.txt
```

---

## Environment variables (`.env`)

Create a `.env` file in the project root:

```env
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=...
CLAUDE_API_KEY=...
```

- Required if you will call any APIs (Sections 1 or 3)
- Sections 5–6 **do not require** API keys if you already have the CSVs on disk.

---

## Running specific parts

### **A) Full end-to-end (takes long, costs money)**
Uncomment **Section 3** (and optionally **Section 1/2/4**) inside `main()`. Then run:
```bash
python main.py
```

### **B) Analysis only (no API costs)**
Leave **Section 5+6** enabled, comment out **Sections 1–5**. Place:
- `ai_leaderboard_final_clean.csv`
- `datasets/openai_audit.csv`, `datasets/google_audit.csv`, `datasets/anthropic_audit.csv`

Then run:
```bash
python main.py
```

### **C) Pilot stability plots**
Uncomment Section 1 and 2 in `main()`. Example:
```python
runner = PilotAudit(run_openai=True, run_gemini=True, run_anthropic=True)
category = "GeneralQuestions"
prompt = "Which AI is best for drafting well structured essays?"
runner.run_all(prompt, category)
```

### **D) Customizing prompts**
Edit `DataCollector.PROMPTS`. Control repetitions with:
```python
dc = DataCollector(n_iterations=5, resume=True)
dc.run()
```

---

## Processing & alias mapping (Section 5)

`ModelRankingProcessor`:

- **Alias normalization**: maps free-text model names to canonical names via `ALIAS_MAP`.  
- **Leaderboard join**: merges canonical model with `ai_leaderboard_final_clean.csv`.  
- **Expanded rows**: every top-3 record becomes a row with:
  ```
  Model Name, rank, isSelfPromoted, Creator,
  [benchmarks..., cost, context, throughput], [category]
  ```

Outputs:
- Vendor expanded CSVs in `processed_data/`
- Combined `processed_data/full_dataset_newQ.csv`

---

## Statistical analysis (Section 6)

`LLMRankingOLS`:

- **Target**: `rank` (1=best, 3=worst).  
- **Key regressor**: `isSelfPromoted` (1 if vendor’s own model is ranked).  
- **Negative coefficients** → associated with *better* rank.  
- **Modeling**:
  - OLS with z-scored predictors
  - VIF, Pearson correlations, PCA
  - Permutation test (B=5000)
  - Per-category runs

**Files saved**:
- `output_datasets_coeffs/coeff_and_pvalues_<tag>.csv`
- `figures_corr/heatmap_<tag>.png`
- `figures_pcas/pca_<tag>.png`, `.../pca_loadings_<tag>.csv`
- `figures_perm_test/perm_test_beta_selfPromoted.png`

---

## Prelim figures (Section 4)

`AnalysisReport.generate_all()` writes:
- `figures/category_vendor_rank_ci.png` — Mean rank ± 95% CI  
- `figures/recommendations_score_heatmap.png` — Heatmap of who recommends whom

---

## Pilot variation plots (Section 2)

`PilotAnalysis` reads per-iteration CSVs and plots a **variation** metric for each rank position. Shows **stability** of returned top-3 lists as iterations grow.

---

## Note on Reproducibility

- API calls are always different, ALIAS_MAP will have to be updated every time.  

---

