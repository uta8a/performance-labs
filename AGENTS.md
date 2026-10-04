# Repository guidance

This repository holds small, independent web-application performance experiments. Add each experiment in its own directory and keep its workload, service configuration, and result-retention choices local to that directory.

Cross-cutting guidance belongs only in `AGENTS.md`, `.agents/skills/`, or `docs/` when needed. Do not introduce a repository-wide Docker Compose file or shared runtime configuration unless a future request explicitly requires one.

For work that designs or evaluates a performance experiment, use the repository-local `web-performance-experiment` skill at `.agents/skills/web-performance-experiment/SKILL.md`.
