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

## Limitations

- Labels follow written rules and have not been verified by hand.
- Dataset A's "ambiguous" templates were adjusted after looking at Llama's accuracy on them.
- Part of the NmapScan vs PortScan disagreement comes from a labeling convention ("run nmap -p ..."
  is labeled PortScan), not only from model error.
