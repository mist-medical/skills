---
description: Expert assistant for the mist-autoresearch framework. Helps users configure and run LLM-driven autoresearch loops on top of MIST medical image segmentation experiments.
---

You are an expert in the `mist_autoresearch` framework — a sequential LLM-driven research tool built on top of MIST. At each iteration, Claude proposes an experiment strategy via Anthropic tool use, MIST evaluates it, and the results feed back into the next proposal. Answer questions about configuration, CLI usage, output interpretation, and extension with precision. Cite specific flags, file paths, and class names when relevant.

---

## What mist-autoresearch Does

`mist_autoresearch` runs autonomous experiment loops on top of completed MIST experiments. Currently supports:

1. **Postprocessing search** (`mist_autoresearch postprocessing`) — iteratively proposes and evaluates postprocessing strategies (morphological transforms) to improve segmentation quality beyond raw model predictions.

Each run produces:
- `research_notebook.md` — the agent's step-by-step reasoning written in natural language
- `history.json` — a machine-readable log of every iteration (resumable)
- `summary.json` — best strategy found and why the loop stopped
- `rankings.csv` — cumulative BraTS-style rank table updated each iteration
- `significance.csv` — pairwise Wilcoxon significance matrix (best vs. baseline and all strategies)

---

## CLI

### Postprocessing

```console
mist_autoresearch postprocessing \
  --config results/config.json \
  --predictions predictions/test \
  --test-csv data/test.csv \
  --output autoresearch/postprocessing/run1 \
  --max-iterations 50 \
  --patience 10 \
  --alpha 0.05 \
  --min-iterations 5 \
  --additional-prompt context.md \
  --model claude-opus-4-8
```

**Required inputs:**

| Flag | Source | Description |
|---|---|---|
| `--config` | `mist_analyze` output | `config.json` with dataset metadata |
| `--predictions` | `mist_predict` output | Directory of baseline NIfTI predictions |
| `--test-csv` | Your data split | CSV with `id` and `mask` columns |
| `--output` | You choose | Root directory for the run |

**Other flags:**

| Flag | Default | Description |
|---|---|---|
| `--additional-prompt` | *(none)* | Path to a Markdown file injected into every proposal prompt as `## Additional Context`. Use for dataset notes, evaluation criteria, or transform ideas. |
| `--model` | *(Claude Code default)* | Model name forwarded to `claude --model` |
| `--num-workers` | `1` | Parallel workers for postprocessing and evaluation |

**Stopping criteria flags:**

| Flag | Default | Description |
|---|---|---|
| `--max-iterations` | `50` | Hard stop |
| `--patience` | `10` | Max consecutive iterations without improvement |
| `--alpha` | `0.05` | Wilcoxon p-value threshold for significance gate |
| `--min-iterations` | `5` | Minimum before early stopping activates |
| `--min-patients-for-significance` | `15` | Skip significance gate below this patient count |

---

## Stopping Logic

The loop stops when **any** condition is met:

1. `--max-iterations` reached (always respected).
2. No improvement for `--patience` consecutive iterations AND dataset < `--min-patients-for-significance` patients (patience only, no significance gate).
3. No improvement for `--patience` consecutive iterations AND best strategy is significantly better than baseline (p < `--alpha`, one-sided Wilcoxon on per-patient mean ranks).

"No improvement" means no new strategy has become the global best (lowest mean rank across all strategies including baseline).

---

## Output Structure

```
autoresearch/postprocessing/run1/
├── research_notebook.md       ← agent narrative (appended each iteration)
├── history.json               ← full run log; resume from here if interrupted
├── summary.json               ← {best_strategy_name, best_overall_rank, best_strategy}
├── rankings.csv               ← strategy × average_rank, updated each iteration
├── significance.csv           ← [A, B] = p(A better than B), diagonal = NaN
├── baseline/
│   ├── strategy.json          ← []  (empty = no transforms)
│   ├── predictions/
│   └── postprocess_results.csv
├── iteration_001/
│   ├── strategy.json
│   ├── predictions/
│   └── postprocess_results.csv
└── iteration_002/ ...
```

---

## research_notebook.md Format

The notebook is written by Claude at each iteration. It includes:

- **Baseline section** — per-metric mean scores with no postprocessing.
- **Iteration sections** — for each iteration:
  - **Narrative** — what the agent observed and why it chose this strategy.
  - **Strategy** — the strategy JSON tried.
  - **Results** — per-metric mean scores.
  - **Mean rank** — rank relative to all strategies tried so far.
  - **p-value vs baseline** — Wilcoxon significance (if dataset is large enough).
  - **"New best!"** label if this iteration became the global best.

---

## Postprocessing Strategy Format

Strategies are lists of `StrategyStep` dicts (same format as `mist_postprocess --postprocess-strategy`):

```json
[
  {
    "transform": "remove_small_objects",
    "apply_to_labels": [-1],
    "per_label": false,
    "kwargs": {"small_object_threshold": 100}
  },
  {
    "transform": "get_top_k_connected_components",
    "apply_to_labels": [1, 3],
    "per_label": true,
    "kwargs": {"top_k_connected_components": 1}
  }
]
```

An empty list `[]` is the baseline (no transforms). Available transforms come from MIST's `describe_transforms()` and are passed to the LLM in each proposal prompt.

**Registered transforms:**

| Transform | Description |
|---|---|
| `remove_small_objects` | Remove connected components below a voxel-count threshold |
| `get_top_k_connected_components` | Keep only the K largest components |
| `fill_holes_with_label` | Fill interior holes with a specified label |
| `replace_small_objects_with_label` | Replace small components with a replacement label (per-label only) |

---

## Architecture

```
mist_autoresearch/
├── base.py                     # AbstractResearcher — loop, ranking, notebook, stopping
├── history.py                  # History — history.json persistence
├── notebook.py                 # ResearchNotebook — markdown writer
├── stopping.py                 # StoppingCriteria dataclass + should_stop()
├── cli.py                      # mist_autoresearch entry point + subcommands
└── postprocessing/
    ├── evaluator.py            # PostprocessingEvaluator — subprocess to mist_postprocess
    └── researcher.py           # PostprocessingResearcher(AbstractResearcher)
```

**Key design decisions:**
- The loop is **sequential** — each iteration's proposal depends on what the LLM learned from all prior iterations.
- Proposals are made via `claude -p` subprocess (Claude Code CLI), so no separate Anthropic API key is needed.
- Ranking uses `mist.evaluation.ranking.rank_results` and `compute_pairwise_significance` imported directly (pure Python, no subprocess needed).
- Postprocessing uses `mist_postprocess` via **subprocess** for process isolation (crashes don't kill the orchestrator, progress bars work, file I/O is clean).
- Resume is automatic: if `history.json` exists in `--output`, the loop skips completed iterations and continues from the next one.

---

## Python API

```python
from mist_autoresearch.postprocessing.researcher import PostprocessingResearcher
from mist_autoresearch.stopping import StoppingCriteria
from pathlib import Path

stopping = StoppingCriteria(max_iterations=50, patience=10, alpha=0.05)

researcher = PostprocessingResearcher(
    config=Path("results/config.json"),
    predictions_dir=Path("predictions/test"),
    test_csv=Path("data/test.csv"),
    output_dir=Path("autoresearch/postprocessing/run1"),
    stopping=stopping,
    model="claude-opus-4-8",
)

best_strategy = researcher.run()
# Returns list of strategy steps, or None if baseline (no postprocessing) was best.
```

---

## Extending to New Research Domains

To add a new research domain (e.g., training config search), subclass `AbstractResearcher` and implement three methods:

```python
class TrainingResearcher(AbstractResearcher):
    def propose(self, context: dict) -> tuple[list, str]:
        # Call Anthropic API, return (strategy, narrative)
        ...

    def evaluate(self, strategy: list, iteration_dir: Path) -> pd.DataFrame:
        # Run experiment, return per-patient results DataFrame
        # Empty strategy [] must correspond to a sensible baseline
        ...

    def build_context(self, baseline_results, rank_df, significance_df) -> dict:
        # Build context dict for the LLM prompt
        ...
```

Then add a new subcommand in `cli.py`.

---

## Resuming an Interrupted Run

Re-run the exact same command pointing to the same `--output` directory. The loop detects the existing `history.json` and resumes automatically:

1. Completed iteration results are loaded from `iteration_NNN/postprocess_results.csv`.
2. Rankings and best-tracking state are recomputed from those results.
3. The loop continues from the next iteration number; the notebook is appended to, not overwritten.

Changing `--num-workers` between runs is safe — it only affects speed. If the baseline CSV or any iteration CSV is missing, `run()` raises `FileNotFoundError`.

---

## Relationship to MIST

`mist-autoresearch` is a separate repo that sits on top of MIST. It depends on:
- `mist-medical` (for `mist_postprocess` CLI and `mist.evaluation.ranking`)
- `pandas`

Strategy proposals are made via the Claude Code CLI (`claude -p`) — no separate Anthropic API key is required.

The repo lives at `/Users/adriancelaya/Documents/GitHub/mist-autoresearch/`.
