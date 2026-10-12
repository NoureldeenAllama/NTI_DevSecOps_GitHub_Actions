# Day 2 Assignment: Build a CI Workflow from Scratch

## Objective
Build a **complete CI workflow on a brand-new repository** from scratch. Each part of this assignment practises one CI/CD skill. When you finish, your workflow will test, package, verify, and report on a small Python project, triggered the right way, with secrets, permissions, artifacts, and caching done properly.

## The Project
Create a new **public** GitHub repository named **`day2-ci-assignment`** and add this starter code. The code is given so you can focus 100% on CI.

`temperature.py`:
```python
def celsius_to_fahrenheit(celsius):
    return celsius * 9 / 5 + 32


def fahrenheit_to_celsius(fahrenheit):
    return (fahrenheit - 32) * 5 / 9


def is_freezing(celsius):
    return celsius <= 0
```

`test_temperature.py`:
```python
from temperature import celsius_to_fahrenheit, fahrenheit_to_celsius, is_freezing


def test_celsius_to_fahrenheit():
    assert celsius_to_fahrenheit(0) == 32
    assert celsius_to_fahrenheit(100) == 212


def test_fahrenheit_to_celsius():
    assert fahrenheit_to_celsius(32) == 0


def test_is_freezing():
    assert is_freezing(-5)
    assert not is_freezing(10)
```

`requirements-dev.txt`:
```text
pytest
```

Run `pytest -v` locally first. It must pass before you write any workflow.

## The Challenge
Create **one** workflow file, `.github/workflows/ci.yml`, plus the extra files named below. Complete the parts in order: commit and push after each part, and check the Actions tab.

### Part 1: Design First
Before writing YAML, create `PIPELINE.md` in the repo answering:
1. Which **events** will trigger your CI, and why?
2. Which **jobs** will you create? Draw the job graph (which run in parallel, which wait for which).
3. What is the **artifact** this pipeline produces?
4. Is this pipeline CI, Continuous Delivery, or Continuous Deployment? Justify in one sentence.
5. What should happen to the `package` job when tests fail, and why is that what you want?

### Part 2: The Skeleton
1. Create `.github/workflows/ci.yml` with `name: Day 2 CI` and a `run-name` that shows who triggered the run (`github.actor`).
2. If the Actions tab shows *"Invalid workflow file"*, read the error, fix the YAML (indentation, colons, quotes), and push again.
3. Add a **status badge** for this workflow at the top of `README.md`.

### Part 3: Triggers
Your workflow must run:
1. On **push** to `main`, **except** when only Markdown files or files in `docs/` changed.
2. On **pull requests** targeting `main`.
3. **Manually**, with two inputs:
   - `python-version`: a **choice** between `"3.12"` and `"3.13"` (default `"3.13"`).
   - `create-package`: a **boolean** (default `true`).
4. On a **schedule**: every Monday at 07:00 **UTC**.

### Part 4: Jobs & Steps
Create these jobs:
| Job | Runs after | Purpose |
|-----|------------|---------|
| `test` | nothing (first) | Run pytest |
| `package` | `test` | Build a `.tar.gz` of the app |
| `verify` | `package` | Check the package works (no checkout allowed!) |
| `report` | `test` and `package`, **always** (even on failure or skip) | Print the result of every upstream job |

Rules:
- Every job has a `timeout-minutes`.
- The `test` job has a step that runs **only if a previous step failed** and prints `🚨 Tests failed on <branch>`.
- `report` must run in **every** case: tests passed, tests failed, or `package` skipped.

### Part 5: Use Actions
1. Use `actions/checkout` and `actions/setup-python` (major version tags).
2. The Python version comes from the manual input when present, otherwise `"3.13"`. (Hint: `${{ inputs.python-version || '3.13' }}`.)
3. Add `.github/dependabot.yml` so GitHub Actions versions are checked **weekly**.

### Part 6: Variables, Contexts & Expressions
1. A **workflow-level** `env` variable `APP_NAME: temperature`.
2. The `package` job is **skipped** when the workflow is run manually with `create-package` unchecked. In all other cases it runs (if tests passed).
3. The `package` job computes a version `0.1.<run_number>` in a step with an `id`, writes it to `$GITHUB_OUTPUT`, and exposes it as a **job output** named `version`.
4. The `report` job reads that version through `needs` and writes a **job summary** (`$GITHUB_STEP_SUMMARY`) containing a Markdown table with: app name, version (or `n/a` when there is none), commit (short SHA), event name, and the result of `test` and `package`.
5. No user-controlled value (inputs, PR title, branch name) may be written directly inside a `run:` script with `${{ }}`. Pass them through `env:`.

### Part 7: Secrets & Permissions
1. Top-level `permissions: contents: read`.
2. Create a repository **variable** `REPORT_TITLE` (e.g. `Temperature CI Report`) and use it as the heading of the job summary.
3. Create a repository **secret** `NOTIFY_TOKEN` (any fake value). In `report`, add a step that runs **only if the secret is set** and prints `Notification sent (token length: N)`, without ever printing the token. (Hints: `secrets` can't be used directly in `if:`, so expose a true/false flag through a job-level `env`. YAML trap: `run: echo "...length: ..."` contains `: ` and breaks the YAML. Use a `run: |` block.)
4. Add a job `pr-comment` that runs **only on pull requests**, after `package`, and uses `GITHUB_TOKEN` with the **minimum** permission to comment on the PR: `✅ CI passed. Package version <version> is ready.`

### Part 8: Artifacts & Caching
1. Enable **pip caching** in the `test` job.
2. `test` writes a JUnit report (`pytest -v --junitxml=reports/junit.xml`) and uploads it as artifact `test-report`, **even when tests fail**, kept for **5 days**.
3. `package` creates `dist/temperature-<version>.tar.gz` (containing `temperature.py` and `README.md`) and uploads it as artifact `package`. The upload must **fail** if no file is found.
4. `verify` **downloads** the `package` artifact (no `actions/checkout`!), extracts it, and runs:
   `python3 -c "from temperature import celsius_to_fahrenheit as c; assert c(100) == 212; print('Package OK')"`

### Part 9: Prove It Works
Perform these scenarios and keep links to the runs:
1. **Green run on `main`:** all jobs ✅, job summary visible, two artifacts downloadable.
2. **Docs-only change:** edit only `README.md` and push. CI must **not** run.
3. **Failing PR:** on a branch, break a test (e.g. `== 213`) and open a PR. `test` ❌, the 🚨 step runs, `test-report` is still uploaded, `package` / `verify` / `pr-comment` are skipped, `report` still runs and shows the failure.
4. **Fixed PR:** fix the test on the same branch. Everything turns green and the bot comments with the version. Merge it.
5. **Manual run:** run the workflow from the Actions tab with `python-version = 3.12` and `create-package` unchecked. `test` runs on Python 3.12, `package` and `verify` are ⏭️ skipped, and the summary shows version `n/a`.
