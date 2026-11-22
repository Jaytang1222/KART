# Environment Lock

This document records the fixed software versions and configuration decisions used for all reported experiments. Reproduce these conditions to obtain comparable numbers.

## Software Versions

| Component | Version |
|-----------|---------|
| JDK | OpenJDK **1.8.0** (`java-8-openjdk-amd64`) |
| Maven | **3.6.3** |
| HBase (client JAR) | **2.2.3** (`org.apache.hbase:hbase-client:2.2.3`) |
| HBase (server) | **2.2.3** |
| ZooKeeper | **3.4.10** |
| Hadoop (HDFS) | **2.8.5** |
| Python | **3.8+** |

## Cluster Topology (4 RegionServers)

Experiments use a four-node HBase cluster (1 Master + 4 RegionServers on separate hosts).  
Exact hostnames and IP addresses are withheld for double-blind review.

Install layout per node:

```
<install_root>/envs/
  hadoop-2.8.5/
  hbase-2.2.3/
  zookeeper-3.4.10/
```

HDFS root dir: `hdfs://<namenode>:9000`  
HBase root: `hdfs://<namenode>:9000/hbase-<znode>`  
ZooKeeper client port: `2181`  
HBase Master RPC: `16100`; RegionServer RPC: `16120`

Client configuration (`src/main/resources/config/hbase/hbase-site.xml` and `config/hbase/hbase-site.xml`) must point to the correct ZooKeeper quorum and `zookeeper.znode.parent` before running experiments.

## Experiment Protocol

| Setting | Value |
|---------|-------|
| Cache mode | `warm` (in-process warmup, 2 passes) |
| Timed trials | 10 |
| Reported metrics | t\_plan / t\_exec / t\_e2e at P50 / P95 / P99 |
| Advanced indexes | `KART_ADVANCED_INDEXES=1` |
| Manifests | `tdrive_v1_ready` / `porto_v1_ready` |

## Maven on WSL (Known Issue)

If a Windows `settings.xml` sets `localRepository` to a Windows path (e.g. `C:\Users\...\...`), that path becomes a broken relative path under WSL. The build scripts (`kart.sh rebuild`) always pass `-Dmaven.repo.local=$HOME/.m2/repository` to avoid this.

## Dependency Notes

- `slf4j-log4j12` excluded from `hbase-client`; logging uses SLF4J + Logback + `log4j-over-slf4j`.
- Guava 11.0.2 and Netty arrive transitively from `hbase-client:2.2.3`; no compile conflict observed.
- `doctor` verifies a live `Connection` + `Admin.listTableNames()` when HBase/ZK are running.

## HBase Version Note

The client JAR is `2.2.3`. In some mixed-classpath deployments, `Admin.getClusterStatus()` may report a different server version due to bundled JARs in the data-loading toolchain. Functional correctness (`doctor` / list tables / smoke / queries) was verified in all reported runs.

## LLM Configuration

- Provider: OpenAI-compatible HTTP endpoint (DeepSeek or equivalent)
- Model: `deepseek-flash` (or any model that supports `response_format=json_object`)
- Credentials: stored in `.env` only — **never committed**
- JSON mode: `LLM_JSON_MODE=true` (required for DraftIR generation)
