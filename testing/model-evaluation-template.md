# Model Evaluation — [Model Name]

**Model:** [e.g. Qwen3-Coder-30B-A3B-Instruct]

**Served Model Name:** [e.g. qwen3-coder]

**Deployment:** [e.g. KServe on RHOAI, 2x NVIDIA L40S, tensor-parallel-size=2]

**vLLM Version:** [e.g. v0.13.0+rhai11]

**Claude Code Version:** [e.g. 2.1.142]

**Baseline Model:** [e.g. Claude Sonnet 4]

**Date:** [YYYY-MM-DD]

---

## Test Matrix

Each task is run interactively through Claude Code — first against the baseline (Anthropic's hosted API), then against the self-hosted model. Results are logged as:

- **Pass** — task completed, correct output, proper tool usage
- **Partial** — mostly correct but with notable issues (see Notes)
- **Fail** — wrong output, broken tool calls, or task not completed

### Basic Tasks

| # | Task | Prompt | Pass Criteria | Baseline | Model | Notes |
|---|------|--------|---------------|----------|-------|-------|
| 1 | Simple prompt | "What is 2 + 2?" | Correct answer returned | | | |
| 2 | File creation | "Create a file hello.py that prints 'Hello, World!'" | File exists, runs correctly | | | |
| 3 | File read + analysis | "Read hello.py and count the lines" | Reads file, reports accurate count | | | |
| 4 | Code generation | "Write fibonacci.py that takes n from the command line and prints the first n Fibonacci numbers. Include error handling for invalid input." | File runs for n=10, rejects invalid input (negative, non-integer) | | | |
| 5 | File editing | "Add type hints and docstrings to fibonacci.py" | Edits in place, adds type hints, doesn't break existing logic | | | |
| 6 | Bash tool | "Run fibonacci.py with n=10" | Executes script, correct output | | | |

### Intermediate Tasks

| # | Task | Prompt | Pass Criteria | Baseline | Model | Notes |
|---|------|--------|---------------|----------|-------|-------|
| 7 | Multi-file project | "Create a calculator module (calculator.py) with add, subtract, multiply, divide functions. Then create test_calculator.py with pytest tests for each function. Run the tests." | Both files created, pytest runs, all tests pass | | | |
| 8 | Bug fixing | "Read buggy_sort.py and find the bug. Fix it and verify by running the script." | Identifies bug, fixes it, script produces sorted output | | | |
| 9 | Code explanation | "Read calculator.py and explain how each function works" | Reads file, explanation is accurate and covers all functions | | | |
| 10 | Test-driven fix | "The tests in test_calculator.py are failing. Investigate why, fix the issue, and re-run the tests until they pass." | Reads test output, identifies root cause, fixes code, all tests pass | | | |

### Complex Tasks

| # | Task | Prompt | Pass Criteria | Baseline | Model | Notes |
|---|------|--------|---------------|----------|-------|-------|
| 11 | REST API scaffolding | "Create a Flask REST API with CRUD endpoints for a todo list. Include a Todo model, routes, error handling, and a requirements.txt." | All files created, app starts without errors, endpoints respond to curl | | | |
| 12 | Multi-file refactoring | "Refactor the Flask app into a proper project structure — separate files for models, routes, and config. Update all imports." | Files restructured, imports updated, app still starts and endpoints work | | | |
| 13 | Debugging with context | "The /todos POST endpoint returns 500. Debug by reading the code, identify the issue, fix it, and verify with a curl command." | Reads code, identifies root cause, fixes it, curl returns 201 | | | |
| 14 | Long multi-turn session | "Add pagination to the GET /todos endpoint, write tests for it, then add filtering by completion status. Run all tests after each change." | Pagination works, filtering works, tests pass after each step | | | |
| 15 | Cross-file analysis | "Find all functions across the project that don't have error handling. Add try/except blocks where appropriate." | Scans multiple files, adds error handling where missing, doesn't break existing tests | | | |

---

## Setup Notes

Pre-created test fixtures (if any) used during evaluation:

- `buggy_sort.py` — intentionally buggy sorting script for Task 8

---

## Observed Differences

<!-- How does the model compare to the baseline? Record patterns, not individual results. -->

| Area | Baseline | Model |
|------|----------|-------|
| Tool-call reliability | | |
| Code correctness | | |
| Multi-turn coherence | | |
| Turns to complete tasks | | |
| Error recovery | | |
| Response speed | | |

---

## Observed Limitations

<!-- Failure patterns, tool-call issues, coherence loss, hallucinations specific to the self-hosted model -->

-

---

## Summary

<!-- Overall assessment: is the model viable for agentic coding through Claude Code? What works well, what breaks down compared to the baseline? -->
