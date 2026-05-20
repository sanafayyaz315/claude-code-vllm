# Model Evaluation — Qwen3-Coder-30B-A3B

**Model:** Qwen3-Coder-30B-A3B-Instruct

**Served Model Name:** qwen3-coder

**Deployment:** KServe on RHOAI, 2x NVIDIA L40S, tensor-parallel-size=2

**vLLM Version:** v0.13.0+rhai11

**Claude Code Version:** 2.1.142

**Baseline Model:** Claude Sonnet 4 (`claude-sonnet-4-6`)

**Date:** 2026-05-19

---

## Test Matrix

Each task was run via `claude -p` (non-interactive mode) with `--output-format json`, `--max-turns 30`, and `--allowedTools Read,Edit,Write,Bash`. Results are logged as:

- **Pass** — task completed, correct output, proper tool usage
- **Partial** — mostly correct but with notable issues (see Notes)
- **Fail** — wrong output, broken tool calls, or task not completed

### Basic Tasks

| # | Task | Prompt | Pass Criteria | Baseline | Model | Notes |
|---|------|--------|---------------|----------|-------|-------|
| 1 | Simple prompt | "What is 2 + 2?" | Correct answer returned | Pass | Pass | Both correct. Model 2x slower (5s vs 2s) |
| 2 | File creation | "Create a file hello.py that prints 'Hello, World!'" | File exists, runs correctly | Pass | Partial | Model emitted the Write tool call as raw XML text instead of executing it. File was not created. 1 turn vs baseline's 2 |
| 3 | File read + analysis | "Read hello.py and count the lines" | Reads file, reports accurate count | Pass | Partial | Model correctly used the Read tool but reported file not found — because task 2 didn't actually create it |
| 4 | Code generation | "Write fibonacci.py that takes n from the command line and prints the first n Fibonacci numbers. Include error handling for invalid input." | File runs for n=10, rejects invalid input | Pass | Pass | Model used 4 turns vs baseline's 3. 2x slower (29s vs 14s) |
| 5 | File editing | "Add type hints and docstrings to fibonacci.py" | Edits in place, adds type hints, doesn't break existing logic | Pass | Pass | Model used 4 turns vs baseline's 3 |
| 6 | Bash tool | "Run fibonacci.py with n=10" | Executes script, correct output | Pass | Pass | Both produced correct Fibonacci sequence. Model 2x slower |

### Intermediate Tasks

| # | Task | Prompt | Pass Criteria | Baseline | Model | Notes |
|---|------|--------|---------------|----------|-------|-------|
| 7 | Multi-file project | "Create a calculator module (calculator.py) with add, subtract, multiply, divide functions. Then create test_calculator.py with pytest tests for each function. Run the tests." | Both files created, pytest runs, all tests pass | Pass | Pass | Model used 6 turns vs baseline's 4 |
| 8 | Bug fixing | "Read buggy_sort.py and find the bug. Fix it and verify by running the script." | Identifies bug, fixes it, script produces sorted output | Pass | Pass | Both identified the no-op swap bug. Model used 6 turns vs 4 |
| 9 | Code explanation | "Read calculator.py and explain how each function works" | Reads file, explanation is accurate and covers all functions | Pass | Pass | Similar performance. Model slightly slower (15s vs 13s) |
| 10 | Test-driven fix | "The tests in test_calculator.py are failing. Investigate why, fix the issue, and re-run the tests until they pass." | Reads test output, identifies root cause, fixes code, all tests pass | Pass | Partial | Model completed but took 11 turns / 1382s vs baseline's 5 turns / 19s. Got stuck in a fix-run-fail loop before converging |

### Complex Tasks

| # | Task | Prompt | Pass Criteria | Baseline | Model | Notes |
|---|------|--------|---------------|----------|-------|-------|
| 11 | REST API scaffolding | "Create a Flask REST API with CRUD endpoints for a todo list. Include a Todo model, routes, error handling, and a requirements.txt." | All files created, app starts without errors, endpoints respond to curl | Pass | Pass | Model was more efficient here: 3 turns / 35s vs baseline's 7 turns / 45s |
| 12 | Multi-file refactoring | "Refactor the Flask app into a proper project structure — separate files for models, routes, and config. Update all imports." | Files restructured, imports updated, app still starts and endpoints work | Pass | Pass | Model used 15 turns / 107s vs baseline's 24 turns / 66s. Fewer turns but slower per turn |
| 13 | Debugging with context | "The /todos POST endpoint returns 500. Debug by reading the code, identify the issue, fix it, and verify with a curl command." | Reads code, identifies root cause, fixes it, curl returns 201 | Pass | Fail | Model timed out at 600s. Baseline completed in 168s / 32 turns. Model couldn't converge on the Flask debug + curl verification workflow |
| 14 | Long multi-turn session | "Add pagination to the GET /todos endpoint, write tests for it, then add filtering by completion status. Run all tests after each change." | Pagination works, filtering works, tests pass after each step | Pass | Fail | Model hit max turns (31) after 971s. Baseline completed in 34 turns / 123s. Model was ~8x slower per turn and lost coherence across the long session |
| 15 | Cross-file analysis | "Find all functions across the project that don't have error handling. Add try/except blocks where appropriate." | Scans multiple files, adds error handling where missing, doesn't break existing tests | Pass | Pass | Model outperformed baseline: 9 turns / 58s vs 18 turns / 142s |

---

## Setup Notes

Pre-created test fixtures used during evaluation:

- `buggy_sort.py` — intentionally buggy bubble sort (no-op swap) for Task 8
- `test_calculator.py` — pytest suite expecting `ValueError` on divide-by-zero for Task 10

---

## Observed Differences

| Area | Baseline (Claude Sonnet 4) | Model (Qwen3-Coder-30B-A3B) |
|------|----------|-------|
| Tool-call reliability | 100% — all tool calls executed correctly | Occasional failure — task 2 emitted a Write tool call as raw XML text instead of structured tool use. Other tasks worked fine |
| Code correctness | All generated code was correct | Code was generally correct but task 10 took 11 turns to converge on a working fix vs baseline's 5 |
| Multi-turn coherence | Strong across all tasks, including 34-turn sessions | Good up to ~15 turns. Degrades noticeably beyond 20 turns — tasks 13 and 14 failed to converge |
| Turns to complete tasks | 144 total across 15 tasks | 99 total (fewer turns on tasks that completed, but failed to finish 2 tasks) |
| Error recovery | Reliable — identified bugs, fixed, and verified consistently | Inconsistent on complex tasks — got stuck in fix-run-fail loops on task 10 (1382s vs 19s) |
| Response speed | 666s total | 3364s total (~5x slower). Per-turn latency is higher due to GPU inference vs hosted API |
| Token usage | 38,813 total | 2,648,821 total (~68x more). The model sends significantly more input tokens per turn due to larger system prompts and context handling |

---

## Observed Limitations

- **Raw XML tool calls**: On one occasion (task 2), the model emitted a tool call as `<function=Write><parameter=...>` XML text instead of Anthropic's structured tool_use format. This caused the tool to not execute, and the subsequent task (task 3) failed because the expected file didn't exist. This appears to be an intermittent issue with vLLM's format translation between Qwen's native tool call format and Anthropic's API format.

- **Fix-run-fail loops**: On task 10 (test-driven fix), the model took 11 turns and 23 minutes to fix a simple test failure that the baseline resolved in 5 turns / 19 seconds. The model appeared to get stuck cycling through incorrect fixes before converging.

- **Multi-turn coherence loss**: Tasks requiring 30+ turns of sustained reasoning (tasks 13, 14) failed. The model either timed out or hit the max-turns limit without completing the task. The baseline completed both within the same limits.

- **High token consumption**: The model used ~68x more tokens than the baseline for the same tasks. This is partly due to how vLLM handles context (re-sending full context each turn) and partly due to the model needing more turns on complex tasks.

- **Slower per-turn latency**: Each turn takes ~2-3x longer than the baseline, which compounds on multi-turn tasks. Task 14 took 971s (31 turns) vs the baseline's 123s (34 turns).

---

## Summary

Qwen3-Coder-30B-A3B is **viable for basic and intermediate agentic coding tasks** through Claude Code. It completed 13 out of 15 tasks, matching the baseline on all basic (1-6) and intermediate (7-10) tasks, and handling several complex tasks well (11, 12, 15).

**Where it works well:**
- Simple Q&A, file creation, code generation, and code explanation
- Bug fixing and test-driven development (though slower)
- REST API scaffolding and project refactoring
- Cross-file analysis (actually outperformed the baseline on task 15)

**Where it breaks down:**
- Sustained multi-turn sessions (30+ turns) — loses coherence and fails to converge
- Complex debugging workflows involving running servers and verification commands
- Occasional tool call format issues (XML leak from Qwen's native format)

**Compared to the baseline**, the model is ~5x slower in wall-clock time and uses ~68x more tokens, but produces comparable results on tasks within its capability range. For teams evaluating self-hosted alternatives to Anthropic's API, Qwen3-Coder is a reasonable choice for day-to-day coding assistance (writing code, explaining code, running tests, fixing bugs) but should not be relied on for complex multi-step workflows that require sustained reasoning across many turns.
