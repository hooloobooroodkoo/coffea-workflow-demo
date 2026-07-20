# coffea-workflow-demo

Companion material for the coffea-workflow demo-day presentation. Every example runs the same [AGC CMS open-data ttbar analysis](https://github.com/iris-hep/analysis-grand-challenge) — the analysis code ([`agc/ttbar_analysis.py`](agc/ttbar_analysis.py), three plain functions) never changes between examples, only the workflow file with its `RunConfig`.

## Install

`coffea-workflow` is on PyPI and pulls in coffea ≥ 2026.7.0 automatically. The AGC processor additionally needs `correctionlib` and `vector`:

```bash
pip install coffea-workflow correctionlib vector
```

(`xgboost` is only needed if you enable `use_inference` in the builder params.)

## From analysis to workflow

`ttbar_analysis.py` is a normal AGC ttbar analysis, split into three plain functions — no decorators, no base classes. Knobs you want to control from outside the analysis (file count, ML inference, …) are just keyword arguments:

```python
# ttbar_analysis.py

def get_fileset(with_failure=False, n_files_max_per_sample=2):
    ...            # -> fileset dict
    return fileset

def run_analysis(fileset, executor=None, use_inference=False, use_triton=False):
    run = processor.Runner(executor=executor, ..., use_result_type=True, skipbadfiles=True)
    return run(fileset, TtbarAnalysis(use_inference, use_triton), treename="Events")
    # -> Ok(result) / Err(exception)

def plotting_1(result):
    ...            # -> plots
```

`executor` is injected by the workflow at run time, so the function never knows which facility it's running on. `use_result_type=True` + `skipbadfiles=True` make the Runner return `Ok`/`Err` instead of raising — that's what lets a failed subset be caught and retried without aborting the run. (`with_failure` isn't part of the real analysis — it's a demo-only switch that corrupts one file's URL on purpose, so there's something for the fault-tolerance behavior to catch.)

In the workflow file, each function is mapped to a typed `Step` and wired into a DAG; knobs are passed through `builder_params` — because they're hashed into the step's cache identity, changing one automatically invalidates exactly the caches it affects:

```python
from coffea_workflow import Step, Workflow, Fileset, Analysis, Plotting, RunConfig, ExecutorConfig, run
from coffea_workflow import facilities
from ttbar_analysis import get_fileset, run_analysis, plotting_1

step_fileset = Step(
    name="Fileset_ttbar", step_type=Fileset, builder=get_fileset,
    builder_params={"with_failure": True,        # demo-only — injects the broken replica
                    "n_files_max_per_sample": 2},
    output="fileset_dict",
)
step_analysis = Step(
    name="Analysis_ttbar", step_type=Analysis, builder=run_analysis,
    builder_params={"use_inference": False},
    input="fileset_dict", output="analysis_payload",
)
step_plotting = Step(
    name="Plot_ttbar", step_type=Plotting, builder=plotting_1,
    input="analysis_payload",
)

workflow = Workflow()
workflow.add(step_fileset)
workflow.add(step_analysis, depends_on=[step_fileset])
workflow.add(step_plotting, depends_on=[step_analysis])

result = run(workflow, RunConfig(strategy="by_dataset", facility=facilities.local))
```

This block is identical across every example below — the only thing that changes per facility is `RunConfig`.

## Examples — all in [`agc/`](agc/)

| File | What it shows | Where to run |
|---|---|---|
| [`workflow_local.ipynb`](agc/workflow_local.ipynb) | `FuturesExecutor`, dataset restriction for quick checks, a broken replica costing one subset — partial result + selective retry | anywhere |
| [`workflow_coffea_casa.ipynb`](agc/workflow_coffea_casa.ipynb) | `DaskExecutor` on casa's cluster, worker provisioning (packages + files), same fault tolerance | coffea-casa JupyterLab |
| [`parallel_vs_sequential.ipynb`](agc/parallel_vs_sequential.ipynb) | sequential vs parallel dispatch, timed, subset sizes printed, identical-results check | coffea-casa JupyterLab |
| [`workflow_lxplus.py`](agc/workflow_lxplus.py) | HTCondor workers in an Apptainer image — walkthrough in [`README_lxplus.md`](agc/README_lxplus.md) | CERN lxplus |

Shared by all examples: `ttbar_analysis.py` (fileset / processor / plotting), `utils/`, `nanoaod_inputs.json`, `corrections.json`, `models/`.

## Toy examples — [`showcase/`](showcase/)

A smaller, faster analysis (a single MET histogram over two tiny datasets — no systematics, no ML) for when a full AGC run is too slow to run live. Same concepts, same `Step`/`RunConfig` shape as `agc/`, minimal physics.

| Directory | What it shows |
|---|---|
| [`split_strategy/`](showcase/split_strategy/) | One notebook per split strategy (no split / by dataset / percentage) on a shared analysis |
| [`facilities/`](showcase/facilities/) | local, coffea-casa, and lxplus factories and executor configs, side by side in one notebook |
| [`coffea_casa/`](showcase/coffea_casa/) | `CoffeaCasaFactory` + Dask cluster |
| [`lxplus/`](showcase/lxplus/) | Building the Apptainer image and running with `LxplusFactory` |
| [`optimisation/`](showcase/optimisation/) | Sequential vs parallel dispatch benchmark |

## Running

- Work **from inside `agc/`** (input JSON paths resolve relative to it); run notebooks top to bottom.
- Analysis knobs are set per step via `builder_params`, no code changes: `n_files_max_per_sample=2` → 18 files ≈ 19.5M events (`1` → ≈ 10.5M, `-1` → the full AGC); `use_inference=False` by default; `with_failure=True` is demo-only and deliberately breaks one file replica — that is the fault-tolerance demo, not a bug. Set it to `False` for a clean run.
- Results are cached under `.cache*/`: rerunning loads finished subsets instantly and retries only the failed ones; delete a cache directory to re-execute from scratch. Changing a `builder_params` value invalidates the affected caches automatically.
