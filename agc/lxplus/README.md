# AGC ttbar on CERN lxplus

`LxplusFactory` runs the workflow with HTCondor batch jobs as Dask workers, each inside an Apptainer image. Unlike coffea-casa there is no pre-existing cluster — the factory creates one on demand (`HTCondorCluster` → `scale(N)` → `Client`) and cancels the jobs in `close()` when the run finishes.

The worker environment comes **entirely from the image** — there is no runtime package installation. The AGC processor needs `correctionlib` and `vector` on top of the defaults (add `xgboost` if you enable `use_inference`). `corrections.json` itself is *not* needed on the workers: the processor is built on the driver and shipped with the corrections embedded.

## Step 1 — generate the deployment files (off lxplus)

From this directory (`agc/`):

```bash
python workflow_lxplus.py
```

On any machine that is not lxplus, `preflight()` writes `worker.def` and `run_on_lxplus.sh` here and prints the exact scp/build/run commands. When prompted for extra packages, enter: `correctionlib, vector`.

(Non-interactive alternative:)

```python
from coffea_workflow.facilities import generate_apptainer_def
generate_apptainer_def(output="worker.def", extra_packages=("correctionlib", "vector"))
```

## Step 2 — build the image on lxplus (once)

```bash
scp worker.def <user>@lxplus.cern.ch:~/
ssh <user>@lxplus.cern.ch
condor_submit -interactive          # get a batch node
cp ~/worker.def . && apptainer build --fakeroot worker.sif worker.def
cp worker.sif ~/worker.sif          # keep a copy in AFS
```

## Step 3 — run

Clone this repo on lxplus (the driver needs `ttbar_analysis.py`, `utils/`, and the input JSONs), put the image next to the script, and run from `agc/`:

```bash
git clone <this-repo> && cd coffea-workflow-demo/agc
cp ~/worker.sif .
tmux new -s agc          # keep the driver alive if your SSH session drops
bash run_on_lxplus.sh
```

> **The driver's lifetime is the cluster's lifetime**: if the script's process dies (dropped SSH, closed laptop), its queued/running HTCondor jobs are removed with it. Run inside `tmux`/`screen` for anything longer than a quick test. Worker logs land in `condor_logs/`.

`run_on_lxplus.sh` creates the VOMS proxy, binds the HTCondor and Kerberos configuration into the container, and re-runs `workflow_lxplus.py` inside `worker.sif`. The proxy is forwarded to the workers for remote file access. The final plot is written to `ttbar_4j1b.png`.

Results are cached in `.cache_lxplus/` exactly as in the other examples — a rerun reprocesses only chunks that failed (a flaky storage endpoint, an evicted HTCondor job) and loads everything else from cache.
