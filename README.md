# text2sql-qlora

QLoRA fine-tune of Qwen2.5-Coder-3B-Instruct for text-to-SQL, with a before/after evaluation on 300 held-out questions.

**Model:** https://huggingface.co/Varshitha2603/qwen2.5-coder-3b-text2sql-qlora

## Results (300 held-out examples)
| Model | Strict exact match | Normalized exact match |
|---|---|---|
| Base (Qwen2.5-Coder-3B-Instruct, prompted) | 0.023 | 0.570 |
| QLoRA fine-tuned | 0.767 | 0.773 |

- **Strict** compares lowercased, whitespace-collapsed SQL. The base model scored 0.023 mostly because it wrapped answers in quotes, used single quotes, and changed capitalization of values.
- **Normalized** also strips wrapping quotes, unifies quote style, treats `INNER JOIN` as `JOIN`, and ignores the order of `AND` conditions.
- The fair accuracy gain is the **normalized** one, about 20 points (0.57 to 0.773). The jump in strict score mostly reflects learning the dataset's output format.
- With 300 test questions, the 95% margin on 0.77 is about ±5 points.

## Method
- **Data:** `b-mc2/sql-create-context` (question, table schema, gold SQL). Shuffled with seed 42, first 300 examples held out for testing and never trained on.
- **Model:** `Qwen/Qwen2.5-Coder-3B-Instruct`, loaded in 4-bit NF4.
- **QLoRA:** rank 16, alpha 32, dropout 0.05, applied to all attention and MLP projection layers. 29.9M trainable parameters (0.96% of the model).
- **Training:** 5,000 examples, 1 epoch, learning rate 2e-4 with cosine schedule, effective batch size 16, loss computed on answer tokens only. Trained on a free Colab T4 GPU in about 21 minutes.
- **Training loss:** step 25: loss 0.212 → step 300: loss 0.046 (313 steps total, 1 epoch)

## Error analysis
I manually categorized the first 12 of the 68 normalized misses (not a random sample):

| Category | Count |
|---|---|
| Metric rejects equivalent SQL (`NOT IN` rewrite, numeric vs quoted literal, cosmetic alias) | 3 |
| Ambiguous question, so the aggregate (`COUNT`, `AVG`, `SUM`) is a coin flip | 3 |
| Wrong literal copied from the question (`"Memorial"` vs `"Memorial tournament"`, `"movie"` vs `"movies"`) | 3 |
| Real structural error (missing `DISTINCT`, wrong join aliases and `GROUP BY`) | 2 |
| Schema typo in the gold column name not reproduced | 1 |

Normalized exact match undercounts accuracy, because it rejects SQL that is valid but written differently.

## Limitations
- Exact-match scoring, not execution accuracy. The dataset has no table contents, so queries can't be run. A next step is execution-based evaluation on Spider.
- Gold labels contain ambiguity and typos, which caps achievable scores.
- The error analysis covers 12 of 68 misses.
- Not compared against a prompted frontier model.
- Single training run, one seed, so no variance estimate.

## Reproduce
1. Open `notebooks/` in Google Colab and select a T4 GPU runtime.
2. Run the cells in order: install, data split, baseline, training, evaluation.
3. Predictions are saved in `results/base_preds.json` and `results/ft_preds.json`.