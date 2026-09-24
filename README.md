# agentic-code-migrator

A Robust Agentic System for Migration of Codebases.

The system migrates legacy code in two regimes: **Python 2 → Python 3** and **Python → Java**. It then _verifies_ that the migration is correct, instead of just checking that it compiles:

1. Static analysis breaks the repository into small fragments and orders them bottom-up by call order.
2. LLM agents translate each fragment.
3. A test-generation agent writes targeted differential tests.
4. The legacy and migrated fragments run side by side on identical inputs in a Docker sandbox.
5. A repair agent patches any behavioural divergence.

## Architecture and status

|         | component                                                                                   | status                                                                                                                                                                            |
| ------- | ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Layer 0 | Static analysis & decomposition (LibCST / tree-sitter call graphs, skeleton scaffold)       | **done**: toolchain [`analysis/`](analysis/README.md); dependency graph, fragments, migration order and the Layer 1 bundle [`layer0/`](layer0/README.md). Target skeleton pending |
| Layer 1 | Deterministic orchestration (LangGraph state machine, retry bounds, human-in-the-loop gate) | **done**: [`layer1/`](layer1/README.md): scheduling, verdict checks, retries, escalation, human review, crash-resume, assembly of the migrated codebase                           |
| Layer 2 | Agents: Analyzer, Translator, Test-Generation, Verdict, Repair, on Groq models              | **done**: [`layer2/`](layer2/README.md). `--layer2 agents`; a dummy that passes every unit is still the default for testing Layer 1                                               |
| Layer 2 | **Executor & Sandbox** (Docker harness, differential comparator)                            | **done**: [`sandbox/`](sandbox/README.md), plus a Java-side harness for Python→Java differential tests                                                                            |
| Layer 2 | **MCP integration**: every agent tool call goes through an MCP server                       | **done**: `acm-analysis` (Layer 0), `acm-workspace` (file system), `acm-sandbox` (Docker, test frameworks): [`layer2/servers/`](layer2/servers/)                                  |

| phase | scope                                                                      | status                                                                                                                                                |
| ----- | -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1     | Tooling & sandbox setup                                                    | done: sandbox, static-analysis tools, Layer 0 decomposition (3 RepoTransBench pairs), MCP servers, the Java target skeleton (Layer 2's `skeleton.py`) |
| 2     | Agent graph (LangGraph, tiered open-weight models)                         | done: Layer 1 orchestrator and the Layer 2 agents on Groq (`gpt-oss-20b` light, `gpt-oss-120b` heavy, `qwen3.8-27b` tests)                            |
| 3     | Targeted verification loop (test generation, verdicts, trace diffing)      | done: coverage-guided differential tests, deterministic Python↔Java comparison, failure records, repair loop                                          |
| 4     | Benchmarking & ablations (Py2→3 hazard suites, RepoTransBench Python→Java) | —                                                                                                                                                     |

## Repository layout

```
agentic-code-migrator/
├── sandbox/          Phase 1 execution sandbox: Docker images, harness, comparator, acceptance tests
├── analysis/         Layer 0 static-analysis toolchain: pinned image, tool runner, verification
├── layer0/           Layer 0 pipeline: dependency graph, fragments, migration order -> bundle for Layer 1
├── layer1/           Layer 1 orchestrator (LangGraph): schedule -> Layer 2 -> verdict -> commit / retry / escalate
├── layer2/           Layer 2: the agents, their MCP servers and the target skeleton (plus a dummy for testing Layer 1)
├── logs/             run logs for image builds, acceptance runs, benchmark runs, sandbox calls (auto-created)
└── benchmarks/       RepoTransBench dataset, downloaded locally (git-ignored)
```

## Setup

### 1. Prerequisites

- **Git**
- **Python 3.8+**. The sandbox, the analysis toolchain and Layer 0 use only the standard library. Layer 1 needs **Python 3.10+** and the packages in `requirements.txt` (LangGraph), installed into a virtual environment in step 3.
- **Docker Desktop**, with the engine running. `docker version` must show a _Server_ section.
- About **10 GB of free disk** if you run the benchmark: 1.8 GB for the four base images, plus roughly 300–750 MB per benchmark project image.

On Windows, run the commands below in Git Bash. They also work in PowerShell if you swap `mkdir -p` for `mkdir` and download the tarball with a browser.

### 2. Clone

```bash
git clone $linkOfRepo
cd agentic-code-migrator          # run every command from here
```
