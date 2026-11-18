# MIT Supercloud Telemetry Fusion & I/O Reconstruction Pipeline

[![Python](https://img.shields.io/badge/Python-3.10%2B%20(stdlib)-3776AB?logo=python&logoColor=white)](https://python.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)


## Overview

End-to-end pipeline transformi
ng raw MIT Supercloud monitoring data into a 
fused, labeled time-series dataset (`tide_dat
aset.csv`) suitable for AI agent training. Pu
re Python stdlib — no external dependencies
.

---

## Pipeline Architecture

```
slurm-l
og.csv ──┐
                ├─→ Ph
ase 0: Core Data Prep ──→ 46,355 COMPLE
TED jobs (peak 15%)
cpu/*/ts.csv ───┤

gpu/*.csv ──────┼─→ Phase 
1: Telemetry Fusion ──→ tide_dataset.cs
v (12,896 bins × 19 features)
node-data.csv 
──┘
labelled_jobids.csv ──→ Logno
rmal calibration per DNN label
```

---

## P
hase 0 — Core Data Prep

| Step | Operation
 | Input | Output |
|------|-----------|-----
--|--------|
| 0.1 | Minimal column load | `s
lurm-log.csv` (29 cols) | 395,914 rows × 14 
cols |
| 0.2 | Filter COMPLETED (state=3) | 3
95,914 rows | 309,037 jobs |
| 0.3 | Peak sam
pling (15% continuous window) | 309,037 jobs 
| 46,355 jobs |

**Step 0.3 detail:** Sweep-l
ine algorithm computes concurrent job count p
er 5-min bin (O(n+bins)), identifies peak con
currency bin (956 jobs), then selects a conti
nuous 15% window centered on peak. Preserves 
temporal ordering and convoy effects.

**Peak
 window:**
- Start epoch: 1,627,892,794
- End
 epoch: 1,631,761,576
- Duration: 1,074.7 hou
rs (~44.8 days)
- Peak concurrency: 956

---


## Phase 1 — Empirical Telemetry Fusion & 
I/O Reconstruction

### Step 1.1 — I/O Extr
action

| Source | Resolution | Records Found
 | Key Metrics |
|--------|------------|-----
----------|-------------|
| CPU time-series (
`cpu/0000/*-timeseries.csv`) | 10s | 54 jobs 
| ReadMB, WriteMB, I/O rate stats |
| GPU tim
e-series (`gpu/0000/*.csv`) | 100ms | 39 jobs
 | SM util, power, SM-dip fraction |
| Node d
ata (`node-data.csv`) | 5min | 7,830,669 rows
 in window | Lustre RPC, FS latency, load avg
 |

**Note:** Only 54/46,355 jobs had CPU tim
e-series files in the sample — the dataset 
provides time-series for a subset of jobs. GP
U coverage is similarly sparse (39 jobs).

##
# Step 1.2 — Cross-Modal Join (5-min bins)


Sweep-line accumulation: for each job, mark 
all bins from `start_bin` to `end_bin`, accum
ulating:
- `active_jobs` — concurrent job c
ount
- `sum_io_mb_real` — total I/O bytes f
rom CPU time-series
- `io_p95_10s` / `io_max_
10s` — micro-burst I/O rates
- `io_fano_mea
n` — burstiness (Fano factor = variance/mea
n)
- `sm_dip_job_count` — jobs with GPU uti
l <5% for >50% of time (I/O stall signature)

- `gpu_jobs_active` — jobs with GPU allocat
ion
- `lustre_rpc_total` / `fs_latency_mean` 
/ `load_avg_mean` / `mem_free_kb_mean` — fr
om node-data

**Output:** 12,896 bins × 5 mi
n = 1,074.7 hours of system-state sequence.


### Step 1.3 — Calibrated Bandwidth Reconst
ruction

**Physical ceiling:** 4 OSS × bonde
d 100 GbE ≈ 40,000 MB/s

**Method:**
1. Obs
erved bandwidth = `sum_io_mb_real / 300s`, ca
pped at ceiling
2. Calibrate RPC→bandwidth 
ratio from bins with both I/O and RPC data
3.
 For bins with RPC but no direct I/O: `bandwi
dth = RPC / median(RPC/MB) / 300s`
4. Combine
d: prefer observed, fallback to reconstructed


**Lognormal calibration:** Fit lognormal (�
�, σ) to real I/O per DNN model label from `
labelled_jobids.csv`. Found 1 model label wit
h sufficient samples.

### Step 1.4 — Empir
ical Risk Labeling

Multi-signal congestion d
etection:

| Condition | Label |
|-----------
|-------|
| RPC > P95 AND saturation > 70% | 
Severe (2) |
| RPC > P95 AND SM-dip jobs > 0 
| Severe (2) |
| Saturation > 70% AND SM-dip 
jobs > 0 | Moderate (1) |
| Otherwise | Clear
 (0) |

**Results:**
- Severe: 3 bins
- Moder
ate: 0 bins
- Clear: 12,893 bins
- Legacy IO>
800 MB/s threshold hits: 203 bins

---

## Ou
tput Files

| File | Size | Description |
|--
----|------|-------------|
| `tide_dataset.cs
v` | 1.5 MB | 12,896 rows × 19 columns, 5-mi
n binned system-state |
| `provenance.json` |
 3.0 KB | Per-column data source, type (obser
ved/reconstructed/derived), notes |
| `peak_w
indow.json` | 210 B | Time window metadata (s
tart, end, duration, job count, peak concurre
ncy) |
| `label_distribution.csv` | 366 B | D
NN model label counts |
| `pipeline.py` | 8.5
 KB | Full pipeline source (stdlib only) |
| 
`pipeline.log` | 1.2 KB | Execution log |

--
-

## tide_dataset.csv Schema

| Column | Typ
e | Source | Provenance |
|--------|------|--
------|------------|
| `time_epoch` | int | d
erived | 5-min bin start timestamp |
| `time_
bin_idx` | int | derived | Sequential bin ind
ex |
| `active_jobs` | int | slurm-log.csv | 
observed — concurrent job count |
| `sum_io
_mb_real` | float | cpu/timeseries | observed
 — total I/O MB in bin |
| `io_p95_10s` | f
loat | cpu/timeseries | observed — P95 of 1
0s I/O rate |
| `io_max_10s` | float | cpu/ti
meseries | observed — max 10s I/O rate |
| 
`io_fano_mean` | float | cpu/timeseries | obs
erved — mean Fano factor |
| `lustre_rpc_to
tal` | float | node-data.csv | observed — s
um Lustre RPCs |
| `fs_latency_mean` | float 
| node-data.csv | observed — mean FS latenc
y |
| `load_avg_mean` | float | node-data.csv
 | observed — mean load average |
| `mem_fr
ee_kb_mean` | float | node-data.csv | observe
d — mean free memory KB |
| `sm_dip_job_cou
nt` | int | gpu/*.csv | observed — I/O-stal
led GPU jobs |
| `gpu_jobs_active` | int | sl
urm tres_alloc | observed — GPU-allocated j
obs |
| `bandwidth_observed_mbs` | float | cp
u/timeseries | observed — MB/s, capped 40 G
B/s |
| `bandwidth_reconstructed_mbs` | float
 | node-data+cpu | reconstructed — from RPC
 ratio |
| `bandwidth_combined_mbs` | float |
 cpu+node-data | reconstructed — observed p
referred |
| `saturation_ratio` | float | der
ived | reconstructed — bw/ceiling |
| `cong
estion_level` | int | derived | derived — 0
/1/2 severity |
| `legacy_io_threshold` | int
 | derived | derived — bw > 800 MB/s flag |


---

## Key Statistics

| Metric | Value |

|--------|-------|
| Total slurm jobs loaded 
| 395,914 |
| COMPLETED jobs | 309,037 (78%) 
|
| Peak-sampled jobs | 46,355 (15%) |
| Peak
 concurrency | 956 |
| Time-series bins | 12,
896 |
| Window duration | 1,074.7 hours (44.8
 days) |
| Max bandwidth | 9,703.3 MB/s |
| M
ax active jobs in bin | 964 |
| CPU I/O cover
age | 54 / 46,355 jobs (0.12%) |
| GPU stats 
coverage | 39 / 46,355 jobs (0.08%) |
| Node-
data rows in window | 7,830,669 |
| Pipeline 
runtime | 98.2 seconds |

---

## Caveats & K
nown Limitations

1. **Sparse time-series cov
erage:** Only ~0.1% of peak-window jobs have 
CPU/GPU time-series files. Most `sum_io_mb_re
al` values are 0; bandwidth relies heavily on
 RPC-based reconstruction.
2. **Single DNN la
bel:** Only 1 model label had ≥5 I/O sample
s for lognormal fitting. The labeled subset (
`labelled_jobids.csv`) is small.
3. **Low con
gestion detection:** Only 3/12,896 bins flagg
ed severe — expected given sparse I/O data.
 Congestion labels will become more meaningfu
l as time-series coverage increases.
4. **No 
real-time API:** Dataset is offline CSV dumps
 — suitable for training, not live control.

5. **Aggressive anonymization:** No job name
s, paths, or kernel versions — agents must 
reason from resource signals only.

---

## S
uggested Next Steps

1. **Expand time-series 
coverage** — incorporate all CPU/GPU files 
(not just peak-window matches)
2. **Semi-supe
rvised labeling** — use the 54 I/O-profiled
 jobs as seeds for clustering the 46K unlabel
ed jobs
3. **Anomaly detection baseline** —
 compare against threshold alerts from Superc
loud operators
4. **Idle-GPU reclaimer** — 
high-impact agent: detect allocated-but-idle 
GPUs (SM util ≈ 0)
5. **Cross-layer root-ca
use linker** — use preserved job IDs to tra
ce scheduler→node→Lustre→GPU for diagno
stic reasoning


---

## Quick Start

```bash
git clone https://github.com/v0ntr3n/pipeline-summary.git
cd pipeline-summary

# Execute the end-to-end data preparation & telemetry fusion pipeline
bash run_pipeline.sh
# or directly with Python:
python src/pipeline.py
```

> **Downstream Scheduler:** The reconstructed dataset produced by this pipeline feeds directly into [**FIAC (`hpc_io_scheduler`)**](https://github.com/v0ntr3n/hpc_io_scheduler) for Forecast-Informed Adaptive Control of HPC job I/O.
