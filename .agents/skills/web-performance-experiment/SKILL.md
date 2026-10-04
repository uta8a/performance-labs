---
name: web-performance-experiment
description: Design, run, and explain reproducible small web-application performance experiments. Use for baselines, benchmark runs, observability correlation, and before/after comparisons; not for automatic performance tuning.
---

# Web performance experiment

## Purpose

Help a small web-application experiment produce evidence that is repeatable and explainable. The outcome is a measured comparison and an explanation, not automatic optimization. Do not add indexes, caches, infrastructure, or architectural changes until observations support one concrete hypothesis.

## When to use

Use this skill when establishing a baseline, designing a load profile, collecting logs or telemetry during a benchmark, testing one proposed performance change, or comparing results. It applies to ISUCON past problems and focused reproductions of HTTP, database, cache, image-processing, locking, or observability behavior.

Choose the smallest useful instrumentation. A benchmark tool such as `ab` is a valid first client, but choose `wrk`, `vegeta`, `k6`, or another tool when it better fits the workload. Do not assume that nginx, Docker, OpenTelemetry, Grafana, or any particular command is present.

## Core concepts

Treat one benchmark execution as a first-class **benchmark run**:

```text
benchmark run
├── metadata
├── summary result
└── time-series telemetry
```

Give every run a unique `run_id`, for example `20261004T104500Z-a3f91c`. Send it with every benchmark request when the target permits it, for example as `X-Benchmark-Run-Id`. Preserve the received value in access logs and attach it to relevant telemetry. This makes the path below queryable as one experiment.

```text
benchmark run → HTTP requests → access logs → metrics / traces
```

`run_id` identifies an execution; `git_sha` identifies code. Keep them separate: one commit may have many runs, including retries, warmups, and measurements. Record a benchmark timestamp for time-series data. Never substitute a Git commit time for the time at which the benchmark actually ran.

Use a named **profile** for a workload, such as `homepage-c50-30s-keepalive`. Keep both its human-readable name and its concrete parameters. A phase is also explicit: `warmup` and `measurement` must be separate runs (or otherwise separately identifiable), and only measurement runs enter the comparison.

## Benchmark workflow

1. Inspect the target experiment and its existing benchmark and observability setup. Define the question, target URL or workload, profile, initial state, and success signals before changing the system.
2. Start or reset the environment as required, verify health, then run a warmup. Give warmup its own `run_id` and metadata.
3. Create a new `run_id` for each measurement. Capture its metadata before starting the load; pass the identifier through requests; retain the raw benchmark output and raw logs where practical.
4. Collect the summary result and time-series telemetry for the exact measurement window. Relate both to the `run_id`, not merely to an approximate clock range.
5. Repeat the baseline under the same profile enough to assess variability (normally three to five measurement runs). Use the median as the primary representative value and investigate material variance.
6. Form one falsifiable bottleneck hypothesis from the evidence. Make one focused change that tests it; avoid bundling unrelated database, proxy, application, and cache changes.
7. Re-establish the same initial conditions and repeat the measurement set with the unchanged profile. Compare before and after, then explain both the result and any limits on confidence.

For capacity behavior, add a concurrency sweep when useful (for example 1, 10, 50, 100, 200). Look for the saturation point: RPS stops growing while latency or errors rise, and correlate it with CPU, connections, and other available telemetry.

## Benchmark run metadata

Record at least the following for every warmup and measurement run. Extend the metadata when a lab has inputs that can affect the result.

```json
{
  "run_id": "20261004T104500Z-a3f91c",
  "started_at": "2026-10-04T10:45:00Z",
  "phase": "measurement",
  "git_sha": "a83fc219ac10",
  "git_branch": "main",
  "git_dirty": false,
  "profile": "homepage-c50-30s-keepalive",
  "target": "http://localhost/",
  "workload": "GET /",
  "concurrency": 50,
  "duration_seconds": 30,
  "request_count": null,
  "keepalive": true,
  "tool": "ab",
  "tool_version": "<captured version>",
  "environment": "local-docker"
}
```

Use either duration or request count as appropriate, and preserve the other as `null` or omit it. Useful additions include host and container resources, image versions, dataset/reset version, client location, sampling settings, and observability mode. Record whether the working tree is dirty; do not silently compare a dirty baseline with a clean change.

For a small lab, keep a run-local record in or under that lab rather than relying only on a telemetry backend:

```text
<lab>/results/<run_id>/
├── meta.json
├── benchmark.txt
├── percentile.csv
└── alp.txt
```

Keep raw inputs (for example access logs, or a durable reference to them) in addition to derived values such as an ALP or Grafana p95. Decide per lab whether large raw artifacts are committed, ignored, or stored elsewhere; do not discard them merely because a dashboard exists.

## Observability guidance

Use structured access logs when the target can produce them. At minimum retain the actual event time, method, URI or route, status, server-observed response time, response bytes, and benchmark `run_id`. Include additional identifiers only when they are useful and safe to retain.

```json
{
  "time": "2026-10-04T10:45:01+09:00",
  "method": "GET",
  "uri": "/",
  "status": 200,
  "request_time": 0.003,
  "body_bytes": 163,
  "benchmark_run_id": "20261004T104500Z-a3f91c"
}
```

For OpenTelemetry, use the benchmark execution time as the event timestamp. Put code and run identity in attributes or resources, for example `service.version=<git SHA>`, `benchmark.run.id=<run_id>`, and `benchmark.profile=<profile>`. Do not treat commit time as telemetry time.

Keep benchmark results distinct from telemetry:

- Summary result: requests/sec, failed requests, client p50/p95/p99, and transfer rate for a single run.
- Time-series telemetry: CPU, memory, request rate, connections, goroutines, GC, and similar values changing during the run.

Client-observed latency and server-observed request time are not interchangeable. Client latency can include networking, connection setup, queueing, and response transfer. Treat divergence as an investigation clue, not an automatic error.

If both an access-log analyzer such as ALP and an observability pipeline are available, retain the analyzer while validating the newer pipeline. Compare count, status distribution, mean, p50, p95, and p99 over the same `run_id` before trusting a dashboard query as an equivalent replacement.

Account for measurement overhead. Access logging, trace sampling, detailed instrumentation, collectors, and profilers can affect results. Where this effect matters, run an explicit observability-enabled versus observability-disabled comparison under the same profile.

## Comparison rules

Compare only matching measurement phases and, unless the experiment changes one deliberately, matching profile, target/workload, tool and version, Keep-Alive setting, environment, initial state, dataset, and observability mode. State every intentional difference.

Evaluate RED signals together: rate (RPS), errors, and duration (p50/p95/p99). Higher throughput alone is not sufficient evidence of improvement; an RPS gain accompanied by a large p99 regression needs explanation. Use median values over repeated runs, report the run count and variability, and treat instability as a result worth investigating.

Local Docker, OrbStack, and similar single-host setups are primarily for relative before/after analysis. The load generator, system under test, database, collector, and dashboards may contend for the same resources, so avoid presenting an absolute RPS as generally transferable. For higher-fidelity capacity tests, separate the load-generator host and SUT host when the experiment requires it.

## Reporting format

Finish an experiment with a concise, evidence-based report containing:

```text
Comparison: <baseline git SHA / run IDs> vs <change git SHA / run IDs>
Condition: <profile, workload, concurrency, duration or count, Keep-Alive, environment>
Runs and variance: <count, median, spread or notable instability>
Rate: <before> → <after> (<change>)
Latency: p50 <before> → <after>; p95 <before> → <after>; p99 <before> → <after>
Errors: <before> → <after>
Evidence: <log/telemetry observations and correlation by run_id>
Conclusion: <supported conclusion, confidence, and remaining caveats>
```

Say whether the data supports the hypothesis. Do not reduce the conclusion to “it became faster.” Include enough run IDs, conditions, and raw-artifact references for another person to reconstruct the comparison.

## Anti-patterns

- Tuning before recording a baseline or making a hypothesis.
- Reusing a `run_id`, or confusing it with a Git SHA.
- Combining warmup and measurement data.
- Comparing different load profiles, initial states, or instrumentation modes without declaring the difference.
- Keeping only dashboard aggregates and losing the underlying logs or benchmark output.
- Assuming client latency must equal server request time.
- Declaring success after one noisy run or based on RPS alone.
- Introducing Redis, Prometheus, Tempo, eBPF, a profiler, or other infrastructure before it answers a demonstrated analysis need.
- Treating local single-host benchmark figures as portable absolute capacity claims.

## ISUCON usage

Use the same evidence loop for an ISUCON past problem:

```text
baseline → observe → hypothesis → one change → benchmark → compare → explain
```

When a bottleneck exposes a knowledge gap, create a separate minimal reproduction for that one theme (for example an index, N+1 query, connection pool, cache, image operation, HTTP behavior, or lock), learn from it, and return to the original problem. Keep each reproduction independent so it can carry its own profile, runs, and conclusions.
