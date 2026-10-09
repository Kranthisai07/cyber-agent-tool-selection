# Behavioral Tool-Selection Corpus for Cybersecurity Agents

Dataset and code for the paper on building an explainable behavioral tool-selection corpus for
cybersecurity AI agents, with smart-tool training and Neural Input Optimization (NIO)
explainability.

Everything runs from the CSV files. No SSH, no live LLM and no network are required to reproduce
the results (notebook 05 can call a local model, but it also ships its saved predictions).

## Two datasets

- **Dataset A**: 640 queries (`cyber_logs.csv`).
- **Dataset B**: 1500 queries (`cyber_dataset_B.csv`) = the 640 queries of A, unchanged, plus 860
  new queries that are intentionally harder.

## How the data was made

- **Annotated tool** (`tool_ground_truth`): the queries come from template families in a Python
  script written with the help of Claude Code. Each template is tied to one tool using the written
  rules in `label_rules.md`. No language model labels individual queries, and the labels have not
  been verified by hand.
- **Llama prediction** (`tool_predicted`): Llama 3.2 3B running locally with Ollama
  (`llama3.2:3b`), temperature 0, seed 42. The model gets the list of tools for the query's role
  and answers with one tool name.
- **Classifier predictions** (`pred_svm`, `pred_svm_random_cv`): TF-IDF + linear SVM, out of fold
  (each row is predicted by a model that never saw it). See "Two ways of splitting" below.

## Files

| File | What it is |
|---|---|
| `cyber_logs.csv` | Dataset A. Columns: `prompt`, `tool_predicted` (Llama), `tool_ground_truth`, `agent_role`, `difficulty`, `category`, `run_id`. |
| `cyber_dataset_B.csv` | Dataset B (columns below). |
| `label_rules.md` | The written rules that decide each label, including the conventions used. |
| `01_data_collection.ipynb` | How the (prompt, tool) pairs are collected, label counts, Llama baseline on A. |
| `02_svm_training.ipynb` | TF-IDF + SVM and other classifiers on A: 5-fold cross validation, confusion matrix. |
| `03_llama_vs_smarttool.ipynb` | Small comparison of the local LLM and the trained SVM on 10 held-out tasks from A. |
| `04_build_dataset_b.ipynb` | Builds dataset B: new template families, seed 42, automated quality checks. |
| `05_llama_predictions.ipynb` | Llama 3.2 predictions for the 860 new rows. Uses `llama_predictions_B.csv` if Ollama is not running. |
| `06_classifiers_dataset_b.ipynb` | Classifiers on B, both split protocols, tables, learning curve, confusion matrix. Adds the SVM columns. |
| `NIOexplainability.ipynb` | Neural Input Optimization on dataset A (agreement vs disagreement model). |
| `llama_predictions_B.csv`, `dataset_b_results.csv`, `confusion_matrix*.png` | Outputs of the notebooks. |

## Columns of `cyber_dataset_B.csv`

| Column | Meaning |
|---|---|
| `prompt` | The query. |
| `tool_ground_truth` | The annotated (correct) tool. |
| `tool_predicted` | Llama 3.2 3B's choice. |
| `pred_svm` | SVM's choice, out of fold with **template-held-out** folds. |
| `pred_svm_random_cv` | SVM's choice, out of fold with **random** folds. |
| `agent_role` | attacker, defender or shared. |
| `difficulty`, `category` | easy / hard, and the query style. |
| `dataset` | `A` (original 640) or `B` (new 860). |
| `family` | Template id. Rows from one template share a family (for A it is approximate). |

## Two ways of splitting

- **Random 5-fold**: how dataset A was evaluated. Rows from the same template can be in both
  training and test, so scores are very high.
- **Template-held-out 5-fold** (`GroupKFold` on `family`): all rows of a template stay together, so
  the test set has phrasing the model did not see in training.

## Results (dataset B, 1500 rows)

| Method | Protocol | All B | A rows | New rows |
|---|---|---|---|---|
| Llama 3.2 3B | none | 77.5% | 79.4% | 76.0% |
| SVM | random 5-fold | 98.1% | 98.9% | 97.6% |
| SVM | template-held-out | 84.7% | 95.5% | 76.7% |

- Trained on A only and tested on the new rows, the SVM gets 68.7%.
- On the new rows with templates held out, the SVM (76.7%) and Llama (76.0%) are about equal. The
  classifier's large advantage over the local LLM appears when sibling templates are in training.
- Dataset A alone reproduces the earlier numbers: SVM 98.3% overall and 94.5% on hard queries,
  Llama 79.4%.
- Disagreement with the annotated tool: Llama 22.5%, SVM (template-held-out) 15.3%, SVM (random
  folds) 1.9%.

## Run

```
pip install -r requirements.txt
jupyter notebook      # open a notebook, then Kernel > Restart & Run All
```

Order for dataset B: `04`, then `05`, then `06`. Keep all files in the same folder.

## Dataset C (1020 queries, written one at a time)

Datasets A and B are template-based, and a TF-IDF + SVM scores about 98% on them under random
splits. Dataset C replaces templates with prompts written one at a time by ChatGPT, following the
instructions from the project supervisor: realistic requests, varied wording and expertise, some
overlap between tools, no templates, balanced classes, correct labels. A 30-prompt pilot was
reviewed first, then two batches (550 and 440) were generated with feedback from the pilot. The
labels were assigned by ChatGPT and spot-checked; they have not all been verified by hand.

A separate set of 405 prompts written by Claude (one row per distinct request, near-copies
removed) is kept as a **cross-source test set** and never used for training.

| File | What it is |
|---|---|
| `data_c/` | The raw parts: pilot 30, batch 550, batch 440, and the Claude 405 |
| `07_build_dataset_c.ipynb` | Merges the parts, checks them, writes `cyber_dataset_C.csv` and `cyber_testset_claude405.csv` |
| `08_llama_dataset_c.ipynb` | Llama 3.2 3B predictions; uses `llama_predictions_C.csv` unless `RERUN = True` |
| `09_classifiers_dataset_c.ipynb` | Classifiers, Llama comparison, cross-source test, learning curve, agreement table; adds `pred_svm` |

Columns of `cyber_dataset_C.csv`: `prompt`, `tool_ground_truth` (annotated), `tool_predicted`
(Llama 3.2 3B), `pred_svm` (SVM, out of fold, stratified 5-fold, seed 42), `source` (pilot,
batch550, batch440), `row_id`.

**Llama setup for C:** dataset C has no attacker/defender roles, so Llama is shown all 11 tools,
each with a one-line description (the same choice the classifier makes). Temperature 0, seed 42.
In A and B it was shown only the tool names for the query's role.

Results (dataset C, 1020 rows):

| Method | Accuracy | Macro F1 |
|---|---|---|
| Llama 3.2 3B (all 11 tools, with descriptions) | 91.2% | 0.910 |
| SVM, 5-fold | 91.0% | 0.910 |
| SVM, 5-fold, similar prompts in the same fold | 90.9% | 0.909 |
| LogReg / MLP / RandomForest, 5-fold | 90.2% / 87.2% / 80.3% | 0.901 / 0.871 / 0.798 |

- On the Claude test set, trained on C: SVM 95.8%, Llama 92.8%.
- SVM and Llama pick the same tool on 84.2% of rows. Llama is wrong on 90 rows, the SVM on 92,
  and both on 15.
- Llama's weakest tool is ListeningPorts (46%): it often picks PortScan or NmapScan for
  questions about this machine's own listening sockets.
- The SVM learning curve is still rising at 816 training rows (83.6% at 408, 91.0% at 816).

Order for dataset C: `07`, then `08`, then `09`.

## Limitations

- Labels follow written rules and have not been verified by hand.
- Dataset A's "ambiguous" templates were adjusted after looking at Llama's accuracy on them.
- Part of the NmapScan vs PortScan disagreement comes from a labeling convention ("run nmap -p ..."
  is labeled PortScan), not only from model error.
