# Distributed Failure Detectors

A comparison of heartbeat-based failure detection algorithms for distributed systems, evaluated on real infrastructure under controlled network conditions.

## Overview

This project implements and empirically compares four failure detector algorithms:

- **Fixed Timeout** - the classical baseline.
  A peer is declared failed if no heartbeat has been received within a fixed time window.
- **Adaptive Timeout** - the timeout threshold adapts to recent network behavior.
  It is computed as the mean plus `k` standard deviations of recent heartbeat inter-arrival intervals.
- **Phi Accrual** - based on Hayashibara et al.'s Phi Accrual Failure Detector (used in systems like Cassandra and Akka).
  Instead of a binary alive/dead timeout, it computes a continuous suspicion level (phi) from a statistical model of heartbeat arrival times, and flags failure once phi crosses a threshold.
- **Confidence Interval** - a statistical detector developed for this project.
  It computes a confidence interval on the mean heartbeat gap and flags failure once the lower bound of that interval exceeds a threshold.

Each node runs all four detectors simultaneously against the same live heartbeat stream, so their behavior can be compared directly under identical conditions.

## What counts as a "failure" here

Failures in this system are **crash-stop node failures**: a node simply stops sending heartbeats.
In the experiment harness, this is simulated by stopping that node's heartbeat-sending thread (not by killing the whole process), which lets the fault injector trigger crash and recovery at precise, scripted times.

Nodes communicate directly over real UDP sockets in a full mesh (every node heartbeats to every other node).
A separate network simulation layer can inject artificial packet loss, delay, and jitter on top of the real network path, which is used to test whether each detector can tell a genuine crash apart from a congested but still-alive peer.

## How to run

Dependencies: `numpy` and `pyyaml` (no `requirements.txt` is currently checked in, install with `pip install numpy pyyaml`).

The system is designed to run as one process per physical node (see `config_ci_10.yaml` for the 5-node address list; edit `host` values for your environment, or leave them all as `127.0.0.1` with distinct ports for local testing).

On each node:

```
python main.py --config config_ci_10.yaml --node node-1 --start-time <unix_timestamp>
```

`--start-time` lets all nodes begin the scenario sequence at the same wall-clock time.
Each run steps through 11 predefined scenarios (`experiments/scenarios.py`): a stable baseline, single and rolling node crashes, three levels of network congestion (with and without a simultaneous crash), and a short congestion spike-and-recovery. Each node writes its own event log as JSON to `logs/`.

After collecting logs from all nodes (`scripts/collect_logs.py` if running on Chameleon Cloud, or just gather local `logs/` directories), merge and analyze them:

```
python scripts/merge_logs.py
python scripts/run_analysis.py
```

This produces `results/results.csv` with, per detector per scenario: false-positive rate, average detection time, and mistake rate (false positives per minute).

## What this demonstrates

- Implementation of a well-known accrual failure detector (Phi Accrual) alongside classical and custom statistical alternatives, evaluated under the same conditions rather than in isolation.
- Real distributed communication: independent processes on independent (or independently-addressed) nodes exchanging UDP heartbeats, synchronized to a shared start time, rather than a single-process simulation.
- Deliberate separation of "real" node failure (thread/process level) from "simulated" network degradation (packet loss, delay, jitter), so detector accuracy can be attributed to the right cause.
- An end-to-end experimental methodology: scripted fault scenarios, per-node event logging, cross-node log merging, and quantitative metrics (false positive rate, detection latency, mistake rate) rather than just a working demo.
- Deployment onto Chameleon Cloud (an NSF-funded distributed systems research testbed) via an isolated VLAN, with a separate, gitignored config for SSH-based log collection.

## Repository layout

- `common/` - node model, UDP heartbeat sender/listener, event recorder
- `detectors/` - the four failure detector implementations
- `simulation/` - network fault injection (loss/delay/jitter) and crash/recover injection
- `experiments/` - scenario definitions and the per-node experiment runner
- `analysis/` - metrics computation and CSV report generation
- `scripts/` - log collection, merging, and analysis entrypoints for multi-node runs

## Note on the existing README

`README.md` in this repo is a "Configuration Guide" for the YAML config files, useful, but it assumes the reader already knows what the project is. This document is meant to replace/supplement it as the primary landing-page explanation; keep the configuration guide content as a secondary section or a linked doc.
