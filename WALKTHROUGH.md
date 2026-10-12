# Day 2 CI Assignment — The Complete Walkthrough

> **Who is this for?** Someone who has *never* written a GitHub Actions workflow
> before. We go line by line. Every part of `Day2-Assignment.md` is shown, then
> explained like you're five, then answered with the real files in this repo,
> then tied back to the final goal with a progress bar.

---

## 0. Read this first: what even *is* GitHub Actions?

**Normal explanation.**
GitHub Actions is a system built into GitHub that runs commands for you on
GitHub's computers ("runners") whenever something happens in your repository —
a push, a pull request, a schedule, or a button you click. You describe *what*
to run in a YAML file stored at `.github/workflows/<name>.yml`. GitHub reads the
file and executes it.

**ELI5.**
Imagine your code repository is a restaurant kitchen. Every time a cook (you)
finishes a dish (a commit), a robot chef (GitHub Actions) automatically tastes
the dish, wraps it in a box, double-checks the box, and writes a report. You
don't have to ask — you just told the robot once, in a recipe card (the YAML
file), what to do when a new dish arrives. If a dish tastes bad, the robot
refuses to box it.

**Glossary you'll see constantly**

| Term | Meaning | ELI5 |
|------|---------|------|
| **Workflow** | One YAML file in `.github/workflows/` | One recipe card |
| **Event** | What starts the workflow (`push`, `pull_request`…) | The dinner bell |
| **Job** | A group of steps that run together on one runner | One station in the kitchen |
| **Step** | One single action inside a job (a command or a reusable action) | One instruction |
| **Runner** | The fresh virtual machine that runs a job | A clean cutting board per chef |
| **Action** | A prebuilt, reusable step (`actions/checkout`) | A kitchen gadget you rent |
| **Artifact** | A file a workflow saves for later/download | A doggy bag |
| **Context** | Values GitHub gives you (`github.actor`, `needs.test.result`) | The clock and name tags |
| **Expression** | `${{ ... }}` — logic evaluated before the step runs | A tiny calculator |
| **Secret** | An encrypted value (tokens, passwords) | A locked drawer |
| **Variable** | An unencrypted, non-secret config value | A sticky note on the fridge |
| **Cache** | Saved downloads to speed up future runs | A stocked pantry |
| **Permission** | What the workflow's `GITHUB_TOKEN` may do | A printed key card |

---

## Part 1 — Design First

###  The required task

> Before writing YAML, create `PIPELINE.md` in the repo answering:
> 1. Which **events** will trigger your CI, and why?
> 2. Which **jobs** will you create? Draw the job graph (which run in parallel, which wait for which).
> 3. What is the **artifact** this pipeline produces?
> 4. Is this pipeline CI, Continuous Delivery, or Continuous Deployment? Justify in one sentence.
> 5. What should happen to the `package` job when tests fail, and why is that what you want?

###  Explained fully

A **trigger** is the "dinner bell": it decides *when* your robot chef wakes up.
On GitHub, the trigger block is written under the keyword `on:`. Common bells:

- `push` — someone pushed commits to a branch.
- `pull_request` — someone opened or updated a PR.
- `workflow_dispatch` — a human clicked "Run workflow" in the UI.
- `schedule` — an alarm clock, written in **cron** syntax.

A **job** is a whole station. Jobs run on separate machines, so by default they
run **in parallel**. To make one job wait for another you use `needs:`. If
`package` has `needs: test`, it will only start after `test` succeeds.

An **artifact** is a file the workflow produces and uploads, e.g. a `.tar.gz`,
a test report, a screenshot. You can download it from the run page later.

The pipeline type:

- **CI (Continuous Integration)** = test/check every change.
- **CD (Continuous Delivery)** = CI *plus* always-ready-to-deploy packages.
- **CD (Continuous Deployment)** = CD *plus* automatic deployment to production.

**ELI5.**

- Trigger = the dinner bell.
- Job graph = who washes dishes first; you can't box food (package) before
  tasting it (test).
- Artifact = the doggy bag you take home.
- CI vs CD = CI only *checks* the food; CD also *sends it to the table*
  (deploy). We only check and box, so we're CI.

**Example of `needs`.**
Two jobs A and B with no `needs` run at the same time (two ovens). Say
`B: needs: [A]` and B waits until A finishes. That instruction is the whole job
graph in one word.

### ✅ The answer

**File created:** `PIPELINE.md` (repo root).

**Why:** the assignment wants the *design thought through before typing YAML*.
Writing the plan first prevents "build first, think later" spaghetti pipelines.

**What's inside, explained:**

- **Section 1 — Events.** A table lists the four bells: `push` to `main`
  (skipping docs-only changes), `pull_request` to `main`, manual
  `workflow_dispatch`, and the Monday `schedule`. The "why" column is the
  reasoning the assignment asks for.
- **Section 2 — Jobs and the graph.** A text diagram shows the chain
  `test → package → verify`, with `report` (waits for `test` + `package`, runs
  always) and `pr-comment` (waits for `package`, PRs only). It states which are
  serial and which are parallel.
- **Section 3 — Artifact.** Names the two artifacts: `package`
  (`dist/temperature-<version>.tar.gz`) and `test-report` (`reports/junit.xml`).
- **Section 4 — Pipeline type.** Says **Continuous Integration** and justifies
  it in one sentence: it tests/packages/verifies but never deploys.
- **Section 5 — When tests fail.** Says `package` is **skipped**, because
  packaging (and possibly publishing) a known-broken build is dangerous. However, the report will still be generated.

### 📊 Progress

```
Overall: [█░░░░░░░░] 1/9  (11%)  — design done, no automation yet
```

---

## Part 2 — The Skeleton

### 📋 The required task

> 1. Create `.github/workflows/ci.yml` with `name: Day 2 CI` and a `run-name`
>    that shows who triggered the run (`github.actor`).
> 2. If the Actions tab shows *"Invalid workflow file"*, read the error, fix the
>    YAML (indentation, colons, quotes), and push again.
> 3. Add a **status badge** for this workflow at the top of `README.md`.

### 🧠 Explained fully

**Where the file lives matters.** GitHub only looks in `.github/workflows/`.
The leading dot makes `.github` a hidden folder on Linux/macOS.

`name:` is the workflow's display title in the Actions tab. `run-name:` lets you
customize the *title of each run*. `github.actor` is a **context** = "the person
or app that triggered this run".

**YAML is whitespace-sensitive.** Indentation uses spaces, never tabs. A stray
`:` inside an unquoted string can break parsing (we'll hit that again in
Part 7). A status badge is just a special image URL GitHub generates; you paste
it as Markdown `![alt](url)`.

**⚡ ELI5.**
`name:` is the recipe's title written on the card. `run-name:` is what you write
on each day's log book — "cooked by Sam". The badge is a little traffic light on
your README that shows green ✅ if the kitchen passed, red ❌ if not.

**💡 YAML trap example.**

```yaml
run: echo "length: 5"      # BROKEN: the ": " confuses the parser
run: |                     # FIXED: a literal block keeps it as plain text
  echo "length: 5"
```

### ✅ The answer

**File created:** `.github/workflows/ci.yml` (head of the file):

```yaml
name: Day 2 CI

run-name: "Day 2 CI - triggered by ${{ github.actor }}"
```

- `name:` → the workflow title in the Actions sidebar.
- `run-name:` → each run is titled e.g. *"Day 2 CI - triggered by noureldeen"*.
  The `${{ github.actor }}` is an **expression** that GitHub replaces with the
  username. The whole thing is in quotes so the colon/dashes are safe.

**File edited:** `README.md` (top):

```markdown
![Day 2 CI](https://github.com/OWNER/day2-ci-assignment/actions/workflows/ci.yml/badge.svg)
```

- The `![...](...)` syntax is a Markdown image.
- The URL is GitHub's auto-generated badge for *this* workflow file. Replace
  `OWNER/day2-ci-assignment` with your real `owner/repo` or the image 404s.

**About the "Invalid workflow file" step:** it's a real teaching moment. When
YAML is malformed, GitHub refuses to run and shows the parser error with a line
number. Common causes: tabs instead of spaces, a missing quote, or an unescaped
`": "`. We avoided all three; `actionlint` reported **zero errors**.

### 📊 Progress

```
Overall: [██░░░░░░░] 2/9  (22%)  — a valid, empty-ish workflow that can't do work yet
```

---

## Part 3 — Triggers

### 📋 The required task

> Your workflow must run:
> 1. On **push** to `main`, **except** when only Markdown files or files in
>    `docs/` changed.
> 2. On **pull requests** targeting `main`.
> 3. **Manually**, with two inputs:
>    - `python-version`: a **choice** between `"3.12"` and `"3.13"` (default `"3.13"`).
>    - `create-package`: a **boolean** (default `true`).
> 4. On a **schedule**: every Monday at 07:00 **UTC**.

### 🧠 Explained fully

All triggers live under the `on:` key (note: `on` is a YAML boolean in old
parsers, which is why some files write it quoted as `"on"`; GitHub accepts both).

- `push.branches` limits the push bell to certain branches.
- `paths-ignore` = "don't ring the bell if the only changed files match these
  patterns". `**.md` = any Markdown file anywhere; `docs/**` = anything inside
  `docs/`.
- `pull_request.branches` limits PRs by their **target** branch (`main` here),
  not the source branch.
- `workflow_dispatch.inputs` defines a small form in the UI. `type: choice`
  renders a dropdown; `type: boolean` renders a checkbox. `default:` sets the
  starting value.
- `schedule` takes a list of **cron** strings. Cron has 5 fields:
  `minute hour day-of-month month day-of-week`. `0 7 * * 1` = minute 0, hour 7,
  any day, any month, weekday 1 (Monday). All times are **UTC**.

**⚡ ELI5.**
`paths-ignore` is you telling the robot: "If the only thing that changed is a
sticky note (docs), don't bother tasting the food." A cron `0 7 * * 1` is an
alarm that says "every Monday at 7am, go taste everything again — maybe an
ingredient went bad overnight."

**💡 Example.** `paths-ignore: ["**.md"]` + a commit that edits `README.md` only
= **no run**. The same commit edits `README.md` and `temperature.py` = **run**,
because at least one file is *not* ignored.

### ✅ The answer

```yaml
on:
  push:
    branches: [main]
    paths-ignore:
      - "**.md"
      - "docs/**"
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      python-version:
        description: "Python version to use"
        type: choice
        options:
          - "3.12"
          - "3.13"
        default: "3.13"
      create-package:
        description: "Build and verify the package"
        type: boolean
        default: true
  schedule:
    - cron: "0 7 * * 1"
```

Line-by-line:

- `push.branches: [main]` → only pushes to `main` ring the bell.
- `paths-ignore` → Markdown anywhere or anything under `docs/` is ignored.
- `pull_request.branches: [main]` → PRs that *target* `main`.
- `workflow_dispatch.inputs.python-version` → a dropdown `3.12` / `3.13`,
  default `3.13` (quoted strings so YAML doesn't read them as numbers).
- `workflow_dispatch.inputs.create-package` → a checkbox, default checked.
- `schedule` → Mondays 07:00 UTC.

The `docs/README.md` file in this repo exists purely so you can *prove* the
`docs/**` ignore rule by editing and pushing it.

### 📊 Progress

```
Overall: [███░░░░░░] 3/9  (33%)  — the workflow now knows WHEN to run
```

---

## Part 4 — Jobs & Steps

### 📋 The required task

> Create these jobs:
>
> | Job | Runs after | Purpose |
> |-----|------------|---------|
> | `test` | nothing (first) | Run pytest |
> | `package` | `test` | Build a `.tar.gz` of the app |
> | `verify` | `package` | Check the package works (no checkout allowed!) |
> | `report` | `test` and `package`, **always** (even on failure or skip) | Print the result of every upstream job |
>
> Rules:
> - Every job has a `timeout-minutes`.
> - The `test` job has a step that runs **only if a previous step failed** and
>   prints `🚨 Tests failed on <branch>`.
> - `report` must run in **every** case: tests passed, tests failed, or
>   `package` skipped.

### 🧠 Explained fully

A job needs three things: a name (`runs-on:`), and a list of `steps:`. Fresh
runners cost money and can hang, so we set `timeout-minutes` as a safety net —
if a job runs too long, GitHub kills it instead of burning an hour.

`needs:` makes a job wait. If an upstream job fails, downstream jobs are
**skipped** by default. To ignore outcomes, use an `if:` with a **status
function**:

- `success()` — everything before passed (this is the implicit default).
- `failure()` — at least one earlier step/job failed.
- `always()` — run no matter what, even if skipped/failed.
- `cancelled()` — the run was cancelled.

Inside a **step**, `if: failure()` means "run only if an earlier step in *this*
job failed." That's exactly the 🚨 rule.

**⚡ ELI5.**
`timeout-minutes` is a kitchen timer: if the rice never cooks, you stop and tell
the customer instead of waiting forever. `if: failure()` is: "If a dish tasted
bad, yell 🚨 so everyone knows." `if: always()` is: "Even if dinner burned, still
write the report — we want to document the disaster."

### ✅ The answer

**Job `test`** (first, no `needs`):

```yaml
  test:
    name: Test
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      # ... (checkout, python, install, pytest) explained in Parts 5 & 8 ...
      - name: Report test failure
        if: failure()
        env:
          BRANCH: ${{ github.ref_name }}
        run: |
          echo "🚨 Tests failed on ${BRANCH}"
```

- `timeout-minutes: 10` → hard stop after 10 minutes.
- `if: failure()` → this step runs **only** when an earlier step failed.
- `BRANCH: ${{ github.ref_name }}` → the branch name is passed through `env`,
  **not** written into `run:` (Part 6 rule).
- `run: |` → a multi-line block; here just one line, but the `|` keeps us safe
  and consistent.
- `${BRANCH}` is normal **shell** substitution, resolved at runtime by bash.

**Job `package`** (`needs: test`):

```yaml
  package:
    name: Package
    needs: test
    ...
```

- `needs: test` → won't start until `test` passes (test's `failure` or `failure`
  of the chain skips it). We add an extra `if:` in Part 6.

**Job `verify`** (`needs: package`):

```yaml
  verify:
    name: Verify
    needs: package
    ...
```

- `needs: package` → serial chain continues.

**Job `report`** (waits for both, runs always):

```yaml
  report:
    name: Report
    needs: [test, package]
    if: always()
    runs-on: ubuntu-latest
    timeout-minutes: 5
```

- `needs: [test, package]` → list syntax for multiple dependencies.
- `if: always()` → **the key line**. Even if `test` fails and `package` is
  skipped, `report` still runs and writes the summary.

**Job `pr-comment`** is explained in Part 7.

### 📊 Progress

```
Overall: [████░░░░░] 4/9  (44%)  — the skeleton of jobs exists and has safety nets
```

---

## Part 5 — Use Actions

### 📋 The required task

> 1. Use `actions/checkout` and `actions/setup-python` (major version tags).
> 2. The Python version comes from the manual input when present, otherwise
>    `"3.13"`. (Hint: `${{ inputs.python-version || '3.13' }}`.)
> 3. Add `.github/dependabot.yml` so GitHub Actions versions are checked **weekly**.

### 🧠 Explained fully

An **action** is a reusable step someone else wrote. `actions/checkout` copies
your repository onto the runner (otherwise the machine is empty!). `actions/
setup-python` installs a chosen Python and can enable dependency caching.

**Version tags.** Actions have releases like `v4.1.7`. Writing `@v4` (a "major
version tag") means "the latest v4.x". It gets you security fixes automatically
without you editing the file. Writing `@v4.1.7` pins exactly. For a beginner,
major tags are the recommended balance.

**The `||` expression.** `a || b` in GitHub expressions means "use `a` if it's
truthy, otherwise `b`". `inputs.python-version` is only defined for manual runs;
on a push it's empty, so `|| '3.13'` supplies the fallback.

**Dependabot** is GitHub's little helper that watches your dependencies and opens
PRs to bump versions. Pointing it at the `github-actions` ecosystem makes it
watch the actions you use (`@v4` → `@v5`, etc.).

**⚡ ELI5.**
`checkout` = "bring a copy of the recipe book into the kitchen." `setup-python`
= "install the right stove." `@v4` = "get the newest version 4 of this gadget."
`inputs.x || '3.13'` = "if the customer chose a stove, use it; otherwise use the
default stove." Dependabot = "a helper who taps you on the shoulder weekly and
says 'a newer tool exists, want it?'"

### ✅ The answer

**In `.github/workflows/ci.yml`:**

```yaml
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version || '3.13' }}
          cache: pip
          cache-dependency-path: requirements-dev.txt
```

- `uses:` → run a prebuilt action instead of a shell command.
- `@v4` / `@v5` → major version tags.
- `with:` → the action's inputs.
- `python-version: ${{ inputs.python-version || '3.13' }}` → manual choice wins,
  otherwise 3.13.
- `cache: pip` → see Part 8.

`actions/download-artifact@v4` and `actions/upload-artifact@v4` (used later) are
also major-tagged.

**File created:** `.github/dependabot.yml`

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

- `version: 2` → the config format version (required header).
- `package-ecosystem: "github-actions"` → watch Actions, not npm/pip.
- `directory: "/"` → where to look (root, where workflow files effectively live).
- `schedule.interval: "weekly"` → check once a week.

### 📊 Progress

```
Overall: [█████░░░░] 5/9  (56%)  — the workflow can now fetch code and Python
```

---

## Part 6 — Variables, Contexts & Expressions

### 📋 The required task

> 1. A **workflow-level** `env` variable `APP_NAME: temperature`.
> 2. The `package` job is **skipped** when the workflow is run manually with
>    `create-package` unchecked. In all other cases it runs (if tests passed).
> 3. The `package` job computes a version `0.1.<run_number>` in a step with an
>    `id`, writes it to `$GITHUB_OUTPUT`, and exposes it as a **job output**
>    named `version`.
> 4. The `report` job reads that version through `needs` and writes a **job
>    summary** (`$GITHUB_STEP_SUMMARY`) containing a Markdown table with: app
>    name, version (or `n/a` when there is none), commit (short SHA), event name,
>    and the result of `test` and `package`.
> 5. No user-controlled value (inputs, PR title, branch name) may be written
>    directly inside a `run:` script with `${{ }}`. Pass them through `env:`.

### 🧠 Explained fully

**`env` levels.** You can set environment variables at three scopes:
workflow (top), job, and step. Inner scopes inherit and can override outer ones.
`APP_NAME` is set once at the top so every job can use `$APP_NAME`.

**Job conditions.** A job's `if:` controls whether it starts. When you write a
custom `if:`, you **replace** the implicit `success()` check, so you must
re-add it yourself: `needs.test.result == 'success'`. `needs.<job>.result` can
be `success`, `failure`, `cancelled`, or `skipped`.

The rule "skip if manual and `create-package` unchecked, run otherwise":
`gitHub.event_name != 'workflow_dispatch'` is true for pushes/PRs/schedules →
run. For a *manual* run it's false, so we fall back to `inputs.create-package`.
Combined: `event != manual || create-package`.

**Outputs.** A step writing `key=value` to the special file `$GITHUB_OUTPUT`
publishes that key. Give the step an `id`, then read `steps.<id>.outputs.<key>`.
A **job** output maps those up with `outputs:` so *other* jobs can read
`needs.<job>.outputs.<key>`.

**Job summaries.** `$GITHUB_STEP_SUMMARY` is a special file; text appended to it
becomes a rich Markdown panel on the run page. 

**Why not put `${{ }}` of user values in `run:`?** Because expressions are
substituted into the script as raw text *before* bash runs. If a PR title were
`"; rm -rf / #` it could become part of the command — **script injection**. The
fix: pass the value through `env:`, then use the shell variable (`"$TITLE"`),
which bash treats safely as data.

**⚡ ELI5.**
- `env` = a whiteboard everyone in the kitchen can read.
- Job `if:` = a bouncer who decides if this station opens today.
- `$GITHUB_OUTPUT` = a note you drop in a box for the next chef.
- `needs.<job>.outputs` = opening that box.
- `$GITHUB_STEP_SUMMARY` = the end-of-day report taped to the wall.
- Passing through `env` instead of `${{ }}` = *handing someone a sealed note*
  rather than *shouting their words into the microphone* (where they could shout
  their own commands).

**💡 Injection example.**

```yaml
# DANGEROUS
run: echo "Hello ${{ github.event.pull_request.title }}"
# SAFE
env:
  TITLE: ${{ github.event.pull_request.title }}   # data
run: echo "Hello ${TITLE}"                        # quoted shell var
```

### ✅ The answer

**1–2. Workflow `env` + skip condition:**

```yaml
env:
  APP_NAME: temperature
```

```yaml
  package:
    needs: test
    if: ${{ needs.test.result == 'success' && (github.event_name != 'workflow_dispatch' || inputs.create-package) }}
```

- On push/PR/schedule → runs (if `test` succeeded).
- Manual run with the box **checked** → runs.
- Manual run with the box **unchecked** → **skipped** ✅.

**3. Version output:**

```yaml
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - name: Compute version
        id: version
        env:
          RUN_NUMBER: ${{ github.run_number }}
        run: echo "version=0.1.${RUN_NUMBER}" >> "$GITHUB_OUTPUT"
```

- `id: version` → names the step so we can read it later.
- `github.run_number` → an auto-incrementing integer per workflow (1, 2, 3…).
- `run_number` is passed via `env` (it's not user-controlled, but it's tidy and
  consistent, and keeps `run` clean).
- `>> "$GITHUB_OUTPUT"` appends `version=0.1.5` to the outputs file.
- The job-level `outputs:` re-publishes it as `needs.package.outputs.version`.

**4. The summary table:**

```yaml
      - name: Write job summary
        env:
          TITLE: ${{ vars.REPORT_TITLE || 'Day 2 CI Report' }}
          VERSION: ${{ needs.package.outputs.version }}
          SHA: ${{ github.sha }}
          EVENT: ${{ github.event_name }}
          TEST_RESULT: ${{ needs.test.result }}
          PACKAGE_RESULT: ${{ needs.package.result }}
        run: |
          version="${VERSION:-n/a}"
          short_sha="${SHA:0:7}"
          {
            echo "## ${TITLE}"
            echo ""
            echo "| Field | Value |"
            echo "|-------|-------|"
            echo "| App | ${APP_NAME} |"
            echo "| Version | ${version} |"
            echo "| Commit | ${short_sha} |"
            echo "| Event | ${EVENT} |"
            echo "| test | ${TEST_RESULT} |"
            echo "| package | ${PACKAGE_RESULT} |"
          } >> "$GITHUB_STEP_SUMMARY"
```

- Every context value is delivered through `env:` — nothing user-controlled is
  inside the `run:` as `${{ }}`. ✅
- `version="${VERSION:-n/a}"` → shell default: if `VERSION` is empty (package
  skipped) use `n/a`.
- `short_sha="${SHA:0:7}"` → first 7 characters of the commit SHA (bash
  substring `${var:start:length}`).
- The `{ ... } >> file` group sends all the echo lines into the summary file.
- The pipe table becomes a real Markdown table on the run page, headed by the
  repository variable `vars.REPORT_TITLE` (Part 7).

**5. The rule, honoured everywhere.** Scanned the file: no `run:` contains
`${{ }}` for inputs, branch names, PR data, or SHA. All arrive via `env:` and are
used as quoted shell variables.

### 📊 Progress

```
Overall: [██████░░░] 6/9  (67%)  — real logic, outputs, and a report
```

---

## Part 7 — Secrets & Permissions

### 📋 The required task

> 1. Top-level `permissions: contents: read`.
> 2. Create a repository **variable** `REPORT_TITLE` and use it as the heading of
>    the job summary.
> 3. Create a repository **secret** `NOTIFY_TOKEN` (fake value). In `report`, add
>    a step that runs **only if the secret is set** and prints
>    `Notification sent (token length: N)`, without ever printing the token.
> 4. Add a job `pr-comment` that runs **only on pull requests**, after `package`,
>    and uses `GITHUB_TOKEN` with the **minimum** permission to comment on the PR:
>    `✅ CI passed. Package version <version> is ready.`

### 🧠 Explained fully

**`GITHUB_TOKEN`** is an automatic, temporary token GitHub creates for every
run. Its power is limited by the `permissions:` block. **Least privilege** = give
it only what it needs. `contents: read` lets it read code; that's all most jobs
need. Commenting on a PR additionally needs `pull-requests: write` — so only the
`pr-comment` job escalates, and only for itself.

**Variables vs Secrets.** Both live in *Settings → Secrets and variables →
Actions*. Variables (`vars.X`) are visible, for non-sensitive config. Secrets
(`secrets.X`) are encrypted and their values are masked in logs — but **you still
shouldn't print them**.

**The `if` + secret problem.** You *cannot* use `secrets.X` in an `if:` at the
step level in a job context — it isn't reliably available there. The trick: put
`secrets.NOTIFY_TOKEN != ''` in a **job-level `env`** (which *can* read
secrets), producing a plain `"true"`/`"false"` string, then test
`if: env.NOTIFY_SET == 'true'`.

**Length without leaking.** `${#NOTIFY_TOKEN}` gives the *number of characters*,
never the contents. So we prove the secret exists without exposing it.

**gh CLI.** The `gh` command-line tool is preinstalled on GitHub runners and
authenticates with `GH_TOKEN`. `gh pr comment <number> --body "..."` posts a
comment.

**⚡ ELI5.**
`permissions` = the key card you hand the robot: "you may *look* at the code, not
change it." The `pr-comment` job gets a slightly bigger card that adds "and post
one sticky note on the PR." A variable is a sticky note on the fridge (everyone
can read). A secret is a locked drawer (encrypted). We tell the robot: "if the
drawer has something in it, say *'I sent a note (it was N letters long)'* — never
read it aloud." The `if: env.NOTIFY_SET` trick is: the robot checks the drawer
*privately*, then flips a light switch ("set/unset") that the sign on the wall
(`if:`) can actually see.

**💡 The YAML trap, again.**
`run: echo "length: ${#TOKEN}"` contains `": "` and breaks YAML. Using
`run: |` + a plain line fixes it.

### ✅ The answer

```yaml
permissions:
  contents: read
```

Top-level least privilege. Every job inherits read-only unless it opts up.

```yaml
      - name: Write job summary
        env:
          TITLE: ${{ vars.REPORT_TITLE || 'Day 2 CI Report' }}
```

Uses the repository **variable** `REPORT_TITLE` as the summary heading, with a
sensible fallback.

```yaml
    env:
      NOTIFY_SET: ${{ secrets.NOTIFY_TOKEN != '' }}
    steps:
      - name: Notify
        if: env.NOTIFY_SET == 'true'
        env:
          NOTIFY_TOKEN: ${{ secrets.NOTIFY_TOKEN }}
        run: |
          echo "Notification sent (token length: ${#NOTIFY_TOKEN})"
```

- Job-level `env: NOTIFY_SET` reads the secret and stores a boolean **string**.
- The step runs only when that flag is `'true'`.
- `run: |` avoids the `": "` YAML trap.
- `${#NOTIFY_TOKEN}` prints only the length. ✅ Never prints the token.

```yaml
  pr-comment:
    name: PR Comment
    needs: package
    if: ${{ github.event_name == 'pull_request' && needs.package.result == 'success' }}
    runs-on: ubuntu-latest
    timeout-minutes: 5
    permissions:
      contents: read
      pull-requests: write
    steps:
      - name: Comment on PR
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          VERSION: ${{ needs.package.outputs.version }}
        run: |
          gh pr comment "$PR_NUMBER" --body "✅ CI passed. Package version ${VERSION} is ready."
```

- `if:` → only on pull requests **and** only when `package` succeeded.
- Job-level `permissions:` → adds exactly `pull-requests: write` (the minimum to
  comment), overriding the top-level read-only for this job alone.
- `GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}` → authenticates `gh`.
- PR number and version arrive via `env` (comply with Part 6 rule #5).
- The body is the required comment.

**Manual setup you must do once in the repo UI:**
1. Settings → Secrets and variables → Actions → **Variables** → New variable
   `REPORT_TITLE` = `Temperature CI Report`.
2. Same page → **Secrets** → New secret `NOTIFY_TOKEN` = any fake text
   (e.g. `fake-123`).

### 📊 Progress

```
Overall: [███████░░] 7/9  (78%)  — security, least privilege, and bot comments
```

---

## Part 8 — Artifacts & Caching

### 📋 The required task

> 1. Enable **pip caching** in the `test` job.
> 2. `test` writes a JUnit report (`pytest -v --junitxml=reports/junit.xml`) and
>    uploads it as artifact `test-report`, **even when tests fail**, kept for
>    **5 days**.
> 3. `package` creates `dist/temperature-<version>.tar.gz` (containing
>    `temperature.py` and `README.md`) and uploads it as artifact `package`. The
>    upload must **fail** if no file is found.
> 4. `verify` **downloads** the `package` artifact (no `actions/checkout`!),
>    extracts it, and runs:
>    `python3 -c "from temperature import celsius_to_fahrenheit as c; assert c(100) == 212; print('Package OK')"`

### 🧠 Explained fully

**Caching.** `pip` downloads packages from the internet every run, which is
slow. `setup-python`'s `cache: pip` computes a key from your requirements files;
if the key is unchanged, the next run restores the saved downloads from GitHub's
cache instead of re-downloading. Result: faster, cheaper runs.

**JUnit XML.** `--junitxml=reports/junit.xml` makes pytest write a
machine-readable results file (tests, pass/fail, durations). GitHub shows it in
a nice Tests tab.

**`if: always()` for the report.** Because a test failure stops later steps by
default, we mark the *upload* step `always()` so the report is captured even in
the failing scenario the assignment cares about.

**`if-no-files-found`.** For `upload-artifact`:
- `warn` (default) → log a warning, job still succeeds. Good for the test
  report.
- `error` → **fail the job**. Required for the package, so a silently-empty build
  can't pass.

**`verify` without checkout.** Normally you must `checkout` to have any files.
Here the magic is `download-artifact`, which fetches the `.tar.gz` the `package`
job produced. Then `tar -xzf` unpacks it and we import the module from the
extracted folder. This proves the **artifact itself** works, independent of the
source tree — exactly what a real consumer would download.

**⚡ ELI5.**
- Caching = keeping yesterday's groceries so you don't shop again.
- JUnit XML = a written food-taste report card.
- `always()` upload = "even if the food tasted bad, still file the report card."
- `if-no-files-found: error` = "if the doggy bag is empty, sound the alarm."
- `verify` = a food critic who only ever sees the sealed doggy bag (never your
  kitchen) and must confirm the meal is good.

### ✅ The answer

**1. Cache (in `test`):**

```yaml
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version || '3.13' }}
          cache: pip
          cache-dependency-path: requirements-dev.txt
```

`cache: pip` on its own only searches for `requirements.txt` or
`pyproject.toml`. Because our file is named `requirements-dev.txt`, we point
`cache-dependency-path` straight at it — otherwise the step fails with *"No file
… matched to [**/requirements.txt or **/pyproject.toml]"*. Now the cache is keyed
on `requirements-dev.txt`, so unchanged deps are restored instantly.

**2. JUnit + resilient upload (in `test`):**

```yaml
      - name: Run tests
        run: pytest -v --junitxml=reports/junit.xml
```
```yaml
      - name: Upload test report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-report
          path: reports/junit.xml
          if-no-files-found: warn
          retention-days: 5
```

- `--junitxml=reports/junit.xml` → writes the report even if assertions fail.
- `if: always()` → upload happens on pass **or** fail. ✅
- `retention-days: 5` → auto-deleted after 5 days. ✅
- `if-no-files-found: warn` → a missing report doesn't hide the real failure.

**3. Build + strict upload (in `package`):**

```yaml
      - name: Build package
        env:
          VERSION: ${{ steps.version.outputs.version }}
        run: |
          mkdir -p dist
          tar -czf "dist/${APP_NAME}-${VERSION}.tar.gz" temperature.py README.md
          ls -l dist
```
```yaml
      - name: Upload package
        uses: actions/upload-artifact@v4
        with:
          name: package
          path: dist/temperature-*.tar.gz
          if-no-files-found: error
          retention-days: 5
```

- `mkdir -p dist` → make the output folder (harmless if it exists).
- `tar -czf <out.tar.gz> temperature.py README.md` → create a gzipped archive
  containing exactly `temperature.py` and `README.md`. Filename is
  `temperature-0.1.<run_number>.tar.gz` via `${APP_NAME}` + `${VERSION}`.
- `ls -l dist` → print what was built so the log is self-explanatory.
- `if-no-files-found: error` → **fails** the job if no archive was produced. ✅
- `path: dist/temperature-*.tar.gz` → the `*` matches the version number.

**4. Download + verify (in `verify`):**

```yaml
  verify:
    name: Verify
    needs: package
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Download package
        uses: actions/download-artifact@v4
        with:
          name: package
          path: pkg

      - name: Extract and verify
        run: |
          mkdir -p verify
          tar -xzf pkg/temperature-*.tar.gz -C verify
          cd verify
          python3 -c "from temperature import celsius_to_fahrenheit as c; assert c(100) == 212; print('Package OK')"
```

- **No `actions/checkout`** → the job literally cannot see the source; it only
  has the downloaded artifact. ✅
- `download-artifact` fetches `package` into `pkg/`.
- `tar -xzf ... -C verify` extracts into `verify/`.
- `cd verify` → so Python finds `temperature.py`.
- `python3 -c "...assert c(100) == 212..."` → imports the **shipped** module and
  asserts the real behaviour. Prints `Package OK` on success.

### 📊 Progress

```
Overall: [████████░] 8/9  (89%)  — the pipeline produces and independently proves its artifacts
```

---

## Part 9 — Prove It Works

### 📋 The required task

> Perform these scenarios and keep links to the runs:
> 1. **Green run on `main`:** all jobs ✅, job summary visible, two artifacts downloadable.
> 2. **Docs-only change:** edit only `README.md` and push. CI must **not** run.
> 3. **Failing PR:** break a test (`== 213`), open a PR. `test` ❌, 🚨 step runs,
>    `test-report` still uploaded, `package`/`verify`/`pr-comment` skipped,
>    `report` still runs and shows the failure.
> 4. **Fixed PR:** fix the test. Everything green, bot comments the version. Merge.
> 5. **Manual run:** `python-version = 3.12`, `create-package` unchecked. `test`
>    runs on 3.12, `package`/`verify` skipped, summary shows version `n/a`.

### 🧠 Explained fully

This part is **acceptance testing**: you don't write new YAML, you *drive* the
pipeline to prove the whole thing behaves under success, failure, skip, docs-only,
and manual conditions. It ties every feature together into observable outcomes.

**⚡ ELI5.** You built a robot chef; now you act like a restaurant inspector. You
order a good meal (green), leave only a sticky note (docs-only), order something
bad on purpose (failing PR), fix it (green PR), and press the secret "custom
order" button (manual run) to make sure the robot behaves.

### ✅ The answer

No code changes — this is what you *do*, and what each scenario exercises in our
files:

| Scenario | What proves it | Where in the code |
|----------|----------------|-------------------|
| 1. Green `main` | All 5 jobs ✅, summary table, 2 artifacts | normal flow; `report` summary; `upload-artifact` ×2 |
| 2. Docs-only | Workflow does **not** trigger | `on.push.paths-ignore: ["**.md", "docs/**"]` |
| 3. Failing PR | `test` ❌ + 🚨 step + report uploaded; `package`/`verify`/`pr-comment` skipped; `report` runs | `if: failure()`; `if: always()` upload; `needs` auto-skip; `report: if: always()` |
| 4. Fixed PR | All green + bot comment with version | `pr-comment` job + `gh pr comment` |
| 5. Manual `3.12`, no package | `test` on 3.12, `package`/`verify` skipped, version `n/a` | `inputs.python-version \|\| '3.13'`; `package.if` create-package check; `${VERSION:-n/a}` |

**How to run each scenario (practical steps):**

1. Push to `main` → Actions → click the run → check all jobs green, scroll to
   *Summary*, download `package` and `test-report` at the bottom.
2. `git commit` a change to `README.md` only → push → confirm **no** new run
   appears in Actions.
3. `git switch -c break-it` → edit a test to `== 213` → push → open a PR to
   `main` → watch `test` fail, 🚨 appear, `test-report` uploaded, downstream
   skipped, `report` green with `package = skipped`.
4. Fix the test on the same branch → push → PR goes green and the bot comments
   `✅ CI passed. Package version 0.1.N is ready.` → merge.
5. Actions → Day 2 CI → *Run workflow* → choose `3.12`, uncheck
   `create-package` → Run → confirm `test` on 3.12, `package`/`verify` skipped,
   summary shows `Version = n/a`.

### 📊 Progress

```
Overall: [█████████] 9/9  (100%)  — pipeline complete and demonstrably correct
```

---

## Final: how every piece connects to the requirements

```
                    ┌─────────────────────────── .github/workflows/ci.yml ───────────────────────────┐
  TRIGGERS          │ push(main, !md/docs) · pull_request(main) · workflow_dispatch(inputs) · cron     │
                    └───────────────────────────────┬───────────────────────────────────────────────┘
                                                    │
   test ──needs──► package ──needs──► verify         │            (side jobs)
    │  │             │  │                │           │
    │  │             │  └──► upload: package (fail if empty) ──► verify downloads (no checkout)
    │  └──► upload: test-report (always, 5d)          │
    │                                                 ├──► report   (needs [test,package], if: always())
    └──► if failure(): 🚨 Tests failed on <branch>    │        └─ summary table + vars.REPORT_TITLE + secret notify
                                                      └──► pr-comment (PR only, needs package, pull-requests: write)

  TOP-LEVEL: env APP_NAME=temperature · permissions contents:read
  EXTRAS:    .github/dependabot.yml (weekly) · README badge · PIPELINE.md design · docs/ trigger proof
```

| Assignment requirement | Delivered by |
|------------------------|--------------|
| Design documented first | `PIPELINE.md` |
| Workflow name + run-name with actor | `ci.yml` lines 1–3 |
| Badge in README | `README.md` |
| 4 trigger types | `ci.yml` `on:` block |
| 5 jobs + graph | `ci.yml` `jobs:` |
| `timeout-minutes` everywhere | every job |
| 🚨 on failure | `test` → `Report test failure` |
| `report` always | `if: always()` |
| Major-tag actions | `@v4` / `@v5` |
| Python from input or 3.13 | `${{ inputs.python-version \|\| '3.13' }}` |
| Dependabot weekly | `.github/dependabot.yml` |
| `APP_NAME` env | `ci.yml` top `env:` |
| Skip package condition | `package.if` |
| Version output | `version` step + job `outputs` |
| Summary table + n/a | `report` → `Write job summary` |
| No user values in `run` | all through `env:` |
| Least privilege | `permissions: contents: read` + job override |
| `REPORT_TITLE` variable | summary heading |
| `NOTIFY_TOKEN` secret (length only) | `report` → `Notify` |
| `pr-comment` job | `ci.yml` `pr-comment` |
| pip cache | `setup-python.cache: pip` |
| JUnit report artifact (always, 5d) | `test` upload |
| Package artifact (fail if empty) | `package` upload |
| Verify without checkout | `verify` download + extract |

**The one-line story:** a push/PR/schedule/manual click rings the bell → `test`
tastes the code and files a report → `package` boxes it (or is skipped) → `verify`
double-checks the sealed box → `report` writes the wall summary → `pr-comment`
sticks a note on the PR. That is Continuous Integration.
