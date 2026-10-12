# Pipeline Design

## 1. Events that trigger CI, and why

| Event | Trigger | Why |
|-------|---------|-----|
| `push` | to `main`, ignoring `**.md` and `docs/**` | Validate every integration into the main branch. Documentation-only changes cannot break code, so we skip them to save CI minutes. |
| `pull_request` | targeting `main` | Validate proposed changes **before** they are merged, so broken code never lands on `main`. |
| `workflow_dispatch` | manual | Let a human re-run the pipeline on demand with a chosen Python version and the option to skip packaging. Useful for testing and one-off runs. |
| `schedule` | `0 7 * * 1` (Mondays 07:00 UTC) | Catch bit-rot: new dependency releases, runner image changes, or flaky tests even when nobody pushes. |

## 2. Jobs and the job graph

```
        test
         │  (needs: test, success)
         ▼
      package ──────────────► pr-comment   (only on pull_request, needs: package)
         │  (needs: package, success)
         ▼
       verify

   report  (needs: [test, package], if: always())
```

- **test** — first, runs `pytest`.
- **package** — after `test` succeeds; builds the `.tar.gz`.
- **verify** — after `package` succeeds; checks the artifact without checkout.
- **pr-comment** — after `package` succeeds, only on pull requests.
- **report** — runs last, in **parallel** with nothing else; it needs both `test` and `package` and runs `always()`.

`test` → `package` → `verify` is a strict serial chain. `report` waits for `test` and `package` but runs regardless of their outcome. `pr-comment` is independent of `verify`.

## 3. Artifact produced

The pipeline produces two artifacts:

1. **`package`** — `dist/temperature-<version>.tar.gz`, the shippable archive containing `temperature.py` and `README.md`.
2. **`test-report`** — `reports/junit.xml`, the machine-readable JUnit test results (uploaded even when tests fail).

## 4. CI, Continuous Delivery, or Continuous Deployment?

This is **Continuous Integration**. It automatically tests, packages, and verifies every change, but it stops before any environment: nothing is deployed to a server, so it is neither Continuous Delivery nor Continuous Deployment.

## 5. What happens to `package` when tests fail?

`package` is **skipped** (never started). Packaging a failed build is pointless and dangerous: a broken artifact could be published or consumed downstream. By making `package` depend on `test` with success, a red test run guarantees no artifact is built. However, the report will still be generated.
