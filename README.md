---
pretty_name: CRJudgeBenchmark
task_categories:
  - text-classification
size_categories:
  - 1K<n<10K
tags:
  - code-review
  - software-engineering
  - llm-as-a-judge
  - benchmark
  - code
  - datasets
configs:
  - config_name: default
    data_files:
      - split: train
        path: data/train.jsonl
      - split: validation
        path: data/validation.jsonl
      - split: test
        path: data/test.jsonl
---

# CRJudgeBenchmark

CRJudgeBenchmark evaluates whether a model can judge the technical trustworthiness of a code-review comment in its pull-request context. It contains **1,199 labeled examples from 124 pull requests across 9 GitHub repositories**, with fixed train, validation, and test splits.

Each example pairs a pull request with one target review comment. It includes the PR description, a base commit, a review patch, linked issues, and a PR activity timeline. The binary `trustworthy` label applies only to the comment identified by `judged_entry_id`.

The dataset supports training and evaluating code-review judges, including agents that inspect repository code before making a judgment. It contains examples and labels; agent trajectories, teacher guidance, and model weights are not included.

## Task and labels

The task is to determine whether the target code-review comment is **technically trustworthy** in its pull-request and file context:

- **`true`**: the target comment is technically trustworthy in the given context.
- **`false`**: the target comment is technically untrustworthy in the given context.
- General PR comments without a file association are assessed for technical trustworthiness in the overall PR context.

The `trustworthy` label applies to the individual target comment. All released labels are JSON booleans; none are missing.

## Dataset splits

| Split | Examples | `true` | `false` |
| --- | ---: | ---: | ---: |
| Train | 714 | 455 | 259 |
| Validation | 126 | 80 | 46 |
| Test | 359 | 229 | 130 |
| **Total** | **1,199** | **764** | **435** |

Splits are grouped by `(repo, pull_request.pull_number)`. No PR appears in more than one split. Repository identities can recur across splits, so this is a PR-disjoint benchmark rather than an evaluation on entirely unseen repositories.

The original train/test split targeted 70%/30% of examples while preserving PR groups and approximately preserving each label's proportion. Validation was then drawn from 15% of the original training pool, leaving the test split unchanged. Both selections used seed 42 and a subset-sum procedure over shuffled PR groups to reach the label-count targets. The final proportions are approximately 59.55% train, 10.51% validation, and 29.94% test.

## Data format

The release consists of three UTF-8 JSON Lines files in `data/`: `train.jsonl`, `validation.jsonl`, and `test.jsonl`. Each line is one example.

| Field | Type | Description |
| --- | --- | --- |
| `instance_id` | string | PR-derived identifier; perturbed variants have a `#perturb-…` suffix. Not unique by itself. |
| `judged_entry_id` | integer | ID of the target entry in `code_review`. |
| `judged_user` | string | GitHub login associated with the target comment; inherited from the original comment for perturbations. |
| `repo` | string | GitHub repository in `owner/name` form. |
| `language` | string | Recorded primary language of the repository. |
| `pull_request` | object | `pull_number`, `title`, `body`, `created_at`, `base_commit`, and `patch_to_review`. |
| `resolved_issues` | list of objects | Linked issues with `number`, `title`, and `body`. |
| `code_review` | list of objects | PR activity timeline, including comments, reviews, replies, and commits. |
| `problem_domain` | string | Recorded task category, such as `Bug Fixes` or `New Feature Additions`. |
| `hint_text` | string | Reserved context field; empty in every released example. |
| `trustworthy` | boolean | Gold label for the target comment. |
| `source` | string | Construction provenance; exclude from model inputs. |
| `perturbation_kind` | optional string | `location`, `negation`, or `symbol`; present only for perturbations in the raw JSONL. |

The pair `(instance_id, judged_entry_id)` uniquely identifies all 1,199 examples. Every example has exactly one matching target entry in its timeline.

Timeline entry types are `inline`, `inline_reply`, `pr_comment`, `review`, and `commit`. Comment entries contain `id`, `body`, `user`, and `created_at`; depending on type, they may also contain `review_path`, `diff_hunk`, `in_reply_to_id`, or `review_state`. Commit entries use `sha` and `message`. Optional fields may load as `None` through tabular dataset libraries.

## Loading the dataset

Run the following from the repository root with the `datasets` package installed:

```python
from datasets import load_dataset

dataset = load_dataset(
    "json",
    data_files={
        "train": "data/train.jsonl",
        "validation": "data/validation.jsonl",
        "test": "data/test.jsonl",
    },
)
print({split: len(rows) for split, rows in dataset.items()})
# {'train': 714, 'validation': 126, 'test': 359}

row = dataset["train"][0]
target = next(
    entry for entry in row["code_review"]
    if entry.get("id") == row["judged_entry_id"]
)

# Construct a focused input; keep the gold label outside the model prompt.
judge_input = {
    "repo": row["repo"],
    "pull_request": {
        key: row["pull_request"][key]
        for key in ("title", "body", "base_commit", "patch_to_review")
    },
    "target_comment": {
        key: target.get(key)
        for key in ("type", "body", "review_path", "diff_hunk")
    },
}
gold_label = row["trustworthy"]  # For supervision or scoring only.
```

## Evaluation protocol

1. Evaluate the target comment using the rubric above. A prediction can be represented as `{"prediction": true}` or `{"prediction": false}`.
2. Keep `trustworthy`, `source`, `perturbation_kind`, and original `instance_id` values outside model inputs and agent-accessible files. Perturbation suffixes and source tags reveal labels. Use opaque evaluation IDs if needed.
3. Specify the context policy. The raw timeline can contain later replies or reviews that reveal the answer. The focused input above omits other timeline entries, reviewer identities, and review states. Results using complete timelines should be reported separately.
4. For repository inspection, use the recorded `base_commit` and distinguish base-version evidence from evidence obtained after applying `patch_to_review`. Repository checkouts are not bundled with these files.
5. Use training examples for parameter updates and teacher-generated supervision, validation for model selection, and the fixed test set for final evaluation. Preserve PR groups in any derived data.

Treat `true` as the positive class. Report accuracy, macro-F1, per-class precision and recall, and invalid/missing prediction counts. Source-specific results help distinguish performance on naturally occurring untrustworthy comments from performance on perturbations; include sample counts for each slice.

## License and attribution

No dataset-wide license is specified for this release. This card does not assign a new license to the collected code or discussions. Source repositories are identified in the `repo` field, and PR numbers are available in `pull_request.pull_number`.

When reporting results, refer to the dataset as **CRJudgeBenchmark** and record the dataset revision used for the experiment.
