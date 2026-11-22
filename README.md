# KART: Natural-Language Trajectory Queries over Key-Value Stores

Official implementation of KART, submitted to DASFAA 2027.

KART maps natural-language trajectory queries to a typed intermediate representation (IR), combines bounded cost-guided search with offline LLM-assisted rewrite rules and cost-patch calibration, and executes verified HBase scans with full result coverage.

---

## Repository Layout

```
config/                       Cost patches, planner defaults, region catalog, HBase client config
datasets/
  tdrive-query/               Frozen query workloads: E1 / complex / simple / workload / scalability
  porto-query/                Same structure for Porto
experiments/
  adapters/                   Third-party bridges (Bao, ParamTree, DIN-SQL, SAG-MQL, DirectIR)
  suites/                     Offline rewrite rules (dual_action_rules.json)
  workloads/scalability/      Scalability workload packages
  results/                    experiment_design.xlsx only; run outputs are generated locally
scripts/
  kart.sh                     Unified operator entry point
  comparative.sh              E1 + E3 comparative experiment
  ablation.sh                 Ablation experiment
  parameter.sh                Parameter sensitivity (OAT)
  workload.sh                 Workload distribution experiment
  scalability.sh              Scalability experiment
src/                          Java source (Maven project)
```

Formal suite definitions are in `src/main/resources/experiments/suites.yaml`.

---

## Requirements

- **OS:** Ubuntu 20.04 / 22.04 (native Linux or WSL 2). Scripts require Bash.
- **JDK:** OpenJDK 8 (`JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64`)
- **Maven:** 3.6.3+
- **HBase server:** 2.2.3 with ZooKeeper 3.4.10 and Hadoop 2.8.5
- **Python:** 3.8+ (adapter bridges and summary scripts)
- **LLM:** Any OpenAI-compatible endpoint supporting `response_format=json_object`

---

## Installation

```bash
git clone <repository-url>
cd <repository-directory>

export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
./scripts/kart.sh rebuild       # runs mvn package; produces target/kart.jar

pip install requests numpy scikit-learn torch
```

**On WSL:** the build script passes `-Dmaven.repo.local=$HOME/.m2/repository` automatically. Do not use a Windows-path `localRepository` in `settings.xml`.

---

## Configuration

### LLM credentials

Create a `.env` file in the repository root:

```bash
LLM_PROVIDER=deepseek
LLM_BASE_URL=https://api.deepseek.com/v1
LLM_MODEL=deepseek-flash
LLM_API_KEY=<your-key>
LLM_JSON_MODE=true
```

The `.env` file is gitignored. Never commit credentials.

| Variable | Description |
|----------|-------------|
| `LLM_PROVIDER` | `deepseek` (default) or `google` |
| `LLM_BASE_URL` | OpenAI-compatible base URL |
| `LLM_MODEL` | Model name, e.g. `deepseek-flash` |
| `LLM_API_KEY` | API key — keep private |
| `LLM_JSON_MODE` | `true` (required for DraftIR generation) |

### HBase client

Edit `src/main/resources/config/hbase/hbase-site.xml` to point at your ZooKeeper quorum:

```xml
<property>
  <name>hbase.zookeeper.quorum</name>
  <value>your-zookeeper-host</value>
</property>
<property>
  <name>zookeeper.znode.parent</name>
  <value>/hbase</value>
</property>
```

Verify the connection:

```bash
./scripts/kart.sh doctor
```

---

## Datasets

### T-Drive

The raw T-Drive trajectory shards are not included in this repository (redistributability constraints). Download them from the [Microsoft T-Drive dataset](https://www.microsoft.com/en-us/research/publication/t-drive-trajectory-data-sample/) and place the files under `datasets/tdrive/`. Then build the HBase snapshot:

```bash
./scripts/kart.sh run build-snapshot --data datasets/tdrive
```

This produces the `tdrive_v1_ready` catalog manifest under `catalog/` (gitignored).

### Porto

Download the Porto taxi dataset and place the cleaned CSV at `datasets/porto/porto_clean.csv`, then:

```bash
./scripts/kart.sh run build-snapshot --data datasets/porto
```

### Query Workloads

Frozen query workloads for both datasets are included:

```
datasets/tdrive-query/    E1_query.json  complex_query.json  simple_query.json
                          workload_query.json  scalability_query.json
datasets/porto-query/     (same structure)
```

---

## Third-Party Baselines

DIN-SQL, SAG-MQL, Bao, and ParamTree require additional upstream repositories:

```bash
bash experiments/adapters/clone-third-party.sh
bash experiments/adapters/clone-third-party.sh --verify
```

Cloned trees land in `experiments/adapters/upstream/` (gitignored). These adapters transplant the control flow of each baseline onto KART's IR and plan family.

---

## Running Experiments

All experiments require a running HBase cluster, populated `catalog/` manifests, and a configured `.env`.

```bash
export KART_EXP_DATE=YYYYMMDD
```

### Comparative (E1 NL→IR + E3 Query Processing)

```bash
./scripts/comparative.sh --dataset tdrive --date $KART_EXP_DATE
./scripts/comparative.sh --dataset porto  --date $KART_EXP_DATE
```

Outputs: `experiments/results/comparativeE1-{dataset}-{date}.md`, `comparativeE3-{dataset}-{date}.md`.

### Workload Distribution

```bash
./scripts/workload.sh --dataset both --date $KART_EXP_DATE
```

### Ablation

```bash
./scripts/ablation.sh --dataset tdrive --date $KART_EXP_DATE
./scripts/ablation.sh --dataset porto  --date $KART_EXP_DATE
```

Factors: `no_rule_llm` / `no_costmodel_llm` / `no_costmodel` / `no_verify`.

### Parameter Sensitivity (OAT)

Beam width ∈ {1, 3, 5, 7}; max\_candidates ∈ {4, 6, 8, 10}:

```bash
./scripts/parameter.sh --dataset tdrive --date $KART_EXP_DATE
./scripts/parameter.sh --dataset porto  --date $KART_EXP_DATE
```

### Scalability

```bash
./scripts/scalability.sh --dataset both --date $KART_EXP_DATE
```

### Single Suite

```bash
./scripts/kart.sh suite e2e-learned --run-id my-run --cache warm --trials 5
./scripts/kart.sh suite parse       --run-id my-e1  --cache warm --trials 1
```

---

## Health Checks

```bash
./scripts/kart.sh doctor   # JDK + HBase connectivity
./scripts/kart.sh probe    # LLM endpoint
./scripts/kart.sh smoke    # End-to-end oracle check (requires catalog + HBase)
```

---

## Method–Arm Mapping

| Paper method | arm / factor id |
|---|---|
| KART | `kart` |
| FullScan | `fullscan` |
| CBO | `cbo` |
| RBO | `rbo` |
| Bao | `bao` |
| ParamTree | `paramtree` |
| DirectIR | `direct-draftir` |
| DIN-SQL (Spider) | `din-sql-spider` |
| DIN-SQL (Bird) | `din-sql-bird` |
| SAG-MQL | `sag-mql-nofeedback` |

---

## Reported Results

Aggregated results are in `experiments/results/experiment_design.xlsx`. Raw run outputs are not committed; regenerate them with the scripts above.

Experiment protocol: warm cache, 10 timed trials, P50 / P95 / P99 latency reported.

---

## License

This project is licensed under the Apache License 2.0. See `LICENSE` for the full text.
