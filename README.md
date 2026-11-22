# KART: Natural-Language Trajectory Queries over Key-Value Stores

Official implementation of KART, submitted to DASFAA 2027.

KART maps natural-language trajectory queries to a typed intermediate representation (IR), combines bounded cost-guided search with offline LLM-assisted rewrite rules and cost-patch calibration, and executes verified HBase scans with full result coverage.

---

## Repository Layout

```
config/                       Cost patches, planner defaults, region catalog, HBase client config
datasets/
  tdrive/                     T-Drive raw trajectory shards (used to build the HBase snapshot)
  tdrive-query/               Frozen query workloads: E1 / complex / simple / workload / scalability
  porto-query/                Same structure for Porto; raw CSV excluded from git (too large)
docs/
  environment-lock.md         Software versions and experiment protocol
  supported-semantics.md      Supported IR semantics and query types
  spec/                       System requirements, design, and experiment specifications
experiments/
  adapters/                   Third-party bridges (Bao, ParamTree, DIN-SQL, SAG-MQL, DirectIR)
  suites/                     Offline rewrite rules (dual_action_rules.json)
  workloads/scalability/      Scalability workload packages
  results/                    experiment_design.xlsx only; run outputs are generated locally
scripts/
  kart.sh                     Unified operator entry point
  comparative.sh              E1 + E3 comparative experiment (Table 3 / Table 5)
  ablation.sh                 Ablation experiment (Table 6)
  parameter.sh                Parameter sensitivity (OAT, Table 7)
  workload.sh                 Workload distribution experiment (Table 4)
  scalability.sh              Scalability experiment (Table 8)
src/                          Java source (Maven project)
```

Formal suite definitions are in `src/main/resources/experiments/suites.yaml`. All experiment scripts resolve suite IDs from this file.

---

## Requirements

- **OS:** Ubuntu 20.04 / 22.04 (native Linux or WSL 2). Scripts require Bash; do not run `.sh` files directly in Windows CMD.
- **JDK:** OpenJDK 8 (`JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64`)
- **Maven:** 3.6.3+
- **HBase server:** 2.2.3 with ZooKeeper 3.4.10 and Hadoop 2.8.5
- **Python:** 3.8+ (third-party adapter bridges and summary scripts)
- **LLM:** Any OpenAI-compatible HTTP endpoint that supports `response_format=json_object`

---

## Installation

```bash
# Clone the repository
git clone <repository-url>
cd <repository-directory>

# Install Java dependencies and build the JAR
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
./scripts/kart.sh rebuild       # runs mvn package; copies kart.jar to target/

# Install Python dependencies for adapter bridges
pip install requests numpy scikit-learn torch
```

**On WSL:** Maven must use the Linux-local repository. The build script passes `-Dmaven.repo.local=$HOME/.m2/repository` automatically. Do not use a Windows-path `localRepository` in `settings.xml`.

---

## Configuration

### LLM credentials

```bash
cp .env.example .env
# Edit .env and fill in LLM_BASE_URL, LLM_MODEL, LLM_API_KEY
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

Edit `src/main/resources/config/hbase/hbase-site.xml` (and optionally `config/hbase/hbase-site.xml`) to point at your ZooKeeper quorum:

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

The `datasets/tdrive/` shards are included in the repository. Load them into HBase once:

```bash
./scripts/kart.sh run build-snapshot --data datasets/tdrive
```

This produces the `tdrive_v1_ready` catalog manifest under `catalog/` (gitignored; regenerate locally).

### Porto

The raw Porto CSV is too large for git. Download the dataset separately and place it at `datasets/porto/porto_clean.csv`, then:

```bash
./scripts/kart.sh run build-snapshot --data datasets/porto
```

Frozen query workloads for both datasets are already included under `datasets/{tdrive,porto}-query/`.

---

## Third-Party Baselines

DIN-SQL, SAG-MQL, Bao, and ParamTree require additional upstream repositories:

```bash
bash experiments/adapters/clone-third-party.sh
bash experiments/adapters/clone-third-party.sh --verify
```

Cloned trees land in `experiments/adapters/upstream/` (gitignored). These are control-flow transplants that operate on KART's IR and plan family — they are not the original upstream systems reproducing their own SQL/MQL/PostgreSQL pipelines.

---

## Running Experiments

All experiments assume the HBase cluster is running, `catalog/` manifests exist, and `.env` is populated. Set `KART_DATASET` and `KART_EXP_DATE` as needed:

```bash
export KART_EXP_DATE=YYYYMMDD
```

### Table 3 — NL→IR Comparison (E1) + Table 5 — Query Processing (E3)

```bash
./scripts/comparative.sh --dataset tdrive --date $KART_EXP_DATE
./scripts/comparative.sh --dataset porto  --date $KART_EXP_DATE
```

Outputs: `experiments/results/comparativeE1-{dataset}-{date}.md`, `comparativeE3-{dataset}-{date}.md`, `case_study-{dataset}-{date}.md`.

### Table 4 — Workload Distribution

```bash
./scripts/workload.sh --dataset both --date $KART_EXP_DATE
```

### Table 6 — Ablation

```bash
./scripts/ablation.sh --dataset tdrive --date $KART_EXP_DATE
./scripts/ablation.sh --dataset porto  --date $KART_EXP_DATE
```

Ablation factors: `no_rule_llm` / `no_costmodel_llm` / `no_costmodel` / `no_verify`.

### Table 7 — Parameter Sensitivity (OAT)

Beam width ∈ {1, 3, 5, 7}; max\_candidates ∈ {4, 6, 8, 10}:

```bash
./scripts/parameter.sh --dataset tdrive --date $KART_EXP_DATE
./scripts/parameter.sh --dataset porto  --date $KART_EXP_DATE
```

### Table 8 — Scalability

```bash
./scripts/scalability.sh --dataset both --date $KART_EXP_DATE
```

Scalability manifests (`tdrive_x10_ready`, `tdrive_x50_ready`, etc.) must be built before running.

### Single Suite

```bash
./scripts/kart.sh suite e2e-learned --run-id my-run --cache warm --trials 5
./scripts/kart.sh suite parse       --run-id my-e1  --cache warm --trials 1
```

---

## Health Checks

```bash
./scripts/kart.sh doctor   # JDK + HBase connectivity
./scripts/kart.sh probe    # LLM endpoint (requires .env)
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

Aggregated results are in `experiments/results/experiment_design.xlsx`. Raw run outputs are not committed; regenerate them locally using the scripts above with the same `KART_EXP_DATE`.

Experiment protocol: warm cache, 10 timed trials, P50 / P95 / P99 latency reported. See `docs/environment-lock.md` for the full version and configuration record.

---

## License

This project is licensed under the Apache License 2.0. See `LICENSE` for the full text.
