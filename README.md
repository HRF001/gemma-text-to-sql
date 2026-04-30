# gemma-text-to-sql

Fine-tuning Google's Gemma-3-1B model with QLoRA to translate natural language questions into SQL queries.

## 🎯 Project Overview

Text-to-SQL is the task of converting natural language questions into executable SQL queries—a capability behind many modern BI tools (Snowflake Cortex Analyst, Databricks Genie, etc.). This project fine-tunes a small instruction-tuned model (Gemma-3-1B) using QLoRA on 5,000 synthetic text-to-SQL samples, and analyzes how fine-tuning improves SQL generation quality across different query complexity levels.

The project follows a rigorous ML workflow: **baseline evaluation → fine-tuning → comparative analysis**, rather than the typical "train and ship" approach.

## 🔍 Methodology

### 1. Baseline Evaluation (Before Fine-tuning)

I first tested the base Gemma-3-1B model on 4 representative samples spanning different SQL complexity levels:

| Sample | Complexity | Baseline Result |
|--------|------------|-----------------|
| 0 | Single JOIN | ❌ Missed JOIN, ignored second table |
| 100 | Subqueries | ❌ Ignored "MAX" requirement entirely |
| 1000 | Basic SQL (DELETE) | ✅ Correct |
| 5000 | JOIN + Aggregation | ❌ Wrong alias, used LIMIT instead of MIN, hallucinated NULL filter |

**Key finding:** The base model handles simple SQL well but systematically struggles with multi-table joins, nested subqueries, and aggregation semantics.

### 2. Fine-tuning Setup

- **Base model:** `google/gemma-3-1b-it` (1B parameters)
- **Method:** QLoRA (4-bit NF4 quantization + LoRA adapters)
- **LoRA config:** `r=8, alpha=16, dropout=0.05, target_modules="all-linear"`
- **Dataset:** `philschmid/gretel-synthetic-text-to-sql` (5,000 train + 200 eval samples, sampled from 100K)
- **Training:** 1 epoch, batch size 2 × grad accumulation 4 (effective batch size 8), learning rate 2e-4, cosine schedule
- **Hardware:** Google Colab T4 GPU (16GB)
- **Training time:** ~71 minutes

### 3. Results

Final losses with no signs of overfitting:

| Metric | Value |
|--------|-------|
| Training Loss | 0.488 |
| Validation Loss | 0.448 |

### 4. Post-Fine-tuning Comparison

Re-evaluating the same 4 samples after fine-tuning:

| Sample | Complexity | Baseline | Fine-tuned | Improvement |
|--------|------------|----------|------------|-------------|
| 0 | Single JOIN | ❌ Missed JOIN | ✅ Correct JOIN with aliases | **Significant** |
| 100 | Subqueries | ❌ Ignored MAX | ⚠️ Used MAX but on wrong column | Partial |
| 1000 | Basic SQL | ✅ | ✅ | Maintained |
| 5000 | JOIN + Aggregation | ❌ Multiple errors | ✅ Correct (using equivalent subquery form) | **Significant** |

**See [sample_outputs.md](sample_outputs.md) for full input/output comparisons.**

## 💡 Key Insights

1. **Fine-tuning effectively addressed JOIN-related failures** — the model went from systematically missing JOINs to using them correctly with proper aliasing.

2. **Subquery / nested-logic tasks remain hard for 1B-scale models.** Even after fine-tuning, the model struggles with second-order reasoning ("count, then take max"). This is likely a model capacity limit, not a training issue—solving it would require a larger base model (e.g., Gemma-7B).

3. **The fine-tuned model demonstrated real generalization**, not memorization. On sample 5000, it produced a SQL query using a *subquery* form rather than the JOIN form in the ground truth—both are functionally equivalent, showing the model learned the underlying task semantics.

4. **No catastrophic forgetting on simple tasks.** Basic SQL operations remained correct after fine-tuning.

## 🛠️ Tech Stack

- **Frameworks:** `transformers`, `peft`, `trl`, `bitsandbytes`, `datasets`
- **Method:** QLoRA (4-bit NF4 quantization + Low-Rank Adaptation)
- **Training:** `SFTTrainer` from TRL with chat-template-based formatting

## 📁 Repository Structure

- `gemma-text-to-sql.ipynb` — Full training notebook
- `adapter/` — Trained LoRA adapter weights (~XX MB)
- `sample_outputs.md` — Detailed before/after generation comparisons
- `README.md` — This file
- `LICENSE` — MIT License

## 🚀 How to Reproduce

1. Open the notebook in [Google Colab](https://colab.research.google.com/) with a T4 GPU runtime.
2. Add your Hugging Face token to Colab Secrets as `HF_TOKEN`.
3. Request access to Gemma at https://huggingface.co/google/gemma-3-1b-it (instant approval).
4. Run all cells. Total runtime: ~80 minutes (mostly training).

## 🔮 Next Steps

- Run a more rigorous evaluation across 100+ validation samples with execution-accuracy metrics (run generated SQL against actual databases and check result match).
- Experiment with larger base models (Gemma-3-4B / 7B) to see if subquery handling improves.
- Try different LoRA ranks (r=16, r=32) to see if capacity is the bottleneck.
- Compare against a few-shot prompting baseline (no fine-tuning, just better prompts) to isolate the value of training.

## 📖 References

- [Fine-Tune Gemma using Hugging Face Transformers and QLoRA](https://ai.google.dev/gemma/docs/core/huggingface_text_finetune_qlora) — Google's official tutorial this project builds on.
- [QLoRA: Efficient Finetuning of Quantized LLMs](https://arxiv.org/abs/2305.14314)
- [LoRA: Low-Rank Adaptation of Large Language Models](https://arxiv.org/abs/2106.09685)

## 📄 License

MIT License — see [LICENSE](LICENSE) file for details.
