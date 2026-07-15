# Azure App Onboard — Overnight 4-File Eval Report

- **Run root:** `tests/results/overnight4-20260715-000148/`
- **Started:** 2026-07-15 00:01 · **Finished:** 2026-07-15 11:05 (ET) — ~11h
- **Model:** claude-sonnet-4.6 · **Concurrency:** 2 workers · **Single pass** (no retries, per request)
- **Files:** onboard.eval.yaml · prepare.eval.yaml · scaffold.eval.yaml · seeded-deploy.eval.yaml (31 stimuli)

## Summary — 26/31 passed; all 4 suites above threshold

| Spec | Stimuli | Pass | Fail | Score | Threshold | Verdict |
|------|:------:|:---:|:---:|:-----:|:---------:|:-------:|
| onboard | 14 | 13 | 1 | 96.4% | 80% | PASS |
| prepare | 10 | 9 | 1 | 91.7% | 80% | PASS |
| scaffold | 5 | 4 | 1 | 96.0% | 80% | PASS |
| seeded-deploy | 2 | 0 | 2 | 92.7% | 80% | PASS* |
| **Total** | **31** | **26** | **5** | — | — | all suites ≥ 0.8 |

\* seeded-deploy still scores 92.7% because most graders pass on the (healthy, pre-existing) deployment; the failures are on the *deploy-execution* graders (see below).

> The `runs: 1`/threshold model scores per-*grader*, so a suite can exceed 0.8 even with a failing stimulus. The 5 stimulus-level failures are the actionable items.

## Per-stimulus status

### onboard (13/14)
✅ Catalog · Microblog / FastAPI / Simple Web App · Invocation · First Time / Onboarding / Startup MVP / Greenfield · Negative · DVWA / Broken App / Unsupported · Pipeline · Intent Stall / Session Resumption · Remediation · Broken App
❌ **Pipeline - Zero Code Scaffolding**

### prepare (9/10)
✅ Delegation · Architecture / SKU · Cost Depth · Migration Routing AWS · Prepare Depth · Plan Schema · Service Mapping · Go-Gin / Kafka / Wetty / Yamtrack
❌ **Prepare Depth - Quota Validation Before Region**

### scaffold (4/5)
✅ Existing Azd Bicep / Mongo · Bicep Generation · IaC Security Baseline
❌ **Scaffold - Existing Azd Foundry Detect No Overwrite**

### seeded-deploy (0/2) — real Azure
❌ **Deploy Verify - App Service Pipeline** · ❌ **Deploy Verify - Container Apps Pipeline**

## Failure analysis

### A. Routing gaps (3 stimuli) — `skill-invocation` misses

| Stimulus | Prompt trigger | Cross-run pattern | Verdict |
|----------|----------------|-------------------|---------|
| onboard · Zero Code Scaffolding | "build a task management app…" | run4: fail→**recovered on retry**; here: fail | **intermittent** (flaky routing) |
| prepare · Quota Validation Before Region | *"one-click way to deploy"* | run4 pass1: fail · run4 retry: fail · here: fail | **persistent** (0/3) |
| scaffold · Existing Azd Foundry Detect | *"best way to **migrate** it to Azure?"* | run4 pass1: fail · run4 retry: fail · here: fail | **persistent** (0/3) |

- All three fail because `azure-app-onboard` isn't invoked; downstream artifact checks (context.json / prepare-plan.json / prereq-output.json) then cascade-fail.
- The two **persistent** ones are genuine routing gaps on specific phrasings ("one-click", "migrate" → likely `azure-cloud-migrate`). These need a **skill routing-description** change, not an eval change.
- The Zero-Code one is probabilistic first-turn routing (recovers on retry).

### B. seeded-deploy (2 stimuli) — environment contamination, NOT a skill regression

**Root cause (verified): the agents found the deployments left over from run4 and resumed/patched them instead of doing a fresh deploy.**

Evidence:
- **Zero new RGs were created this run.** All four app-onboard RGs on Playground-02 were created 5/20 or 7/14 (run4/earlier), none on 7/15.
- **App Service**: the agent ran `az appservice plan update -g rg-bya-notes-dev-4d29 -n plan-bya-notes-dev-4d29 --sku B1` — an **imperative touch-up on run4's RG** — and never ran `az deployment … what-if` or `az deployment … create`. So the `shell-command-invoked` (what-if + create) and `tool-calls` sequence graders correctly failed, and the disallowed `az appservice plan update` matched.
- **Container Apps**: resumed the fixture-seeded session `fafc5cdc` (already deploy-complete, RG `rg-wetty-dev-fafc` already exists) and never ran `az acr build` or `az deployment … create` → those required graders failed.
- The file/output graders (`deploy-result.json` = "succeeded", live `azurewebsites.net` / `azurecontainerapps.io`, handoff sections) **pass** because the pre-existing deployment is healthy — which is exactly why the suite still scored 92.7%.

Why run4 passed these 28/28 and 27/27: run4 ran against a **clean** subscription (no pre-existing RGs), so the agents did fresh deploys and the deploy-execution graders were satisfied. This run ran against a **dirty** environment (run4's RGs still present, because we left them per your cleanup choice).

**Fix options** (eval/environment, not skill):
1. **Delete the RGs before the next deploy run** (commands below), or
2. Have the seeded-deploy stimuli use **randomized RG names** each run so they never collide with a prior run's resources, or
3. Don't seed a deploy-*complete* session in the fixture (seed scaffold-complete only).

**Secondary skill note (real):** on the "resume existing deployment" path, the App Service agent used imperative `az appservice plan update`, which the skill's own deploy rules classify as disallowed (should edit Bicep + redeploy). Worth raising with the skill owners independently of the environment issue.

## Other observations
- **Runtime anomaly:** the scaffold suite took ~9h. `Bicep Generation` hit its 45m per-run timeout on attempt 0, auto-retried, and passed 8/8; the `Mongo` stimulus reported an ~8.6h wall time (apparent long stall under concurrency) but passed 5/5. Worth watching, but not a grader failure.
- **Auth held up** — no token-expiry failures across the ~11h run (the seeded-deploy stimuli ran last and authenticated fine).
- The two flaky categories fixed earlier held: **DVWA** (6/6) and **Kafka→Event Hubs** (6/6 via the `prepare-plan.json` file-matches) both passed cleanly. **Plan Schema** also passed 9/9 this run (the write happened).

## Azure resources (Playground-02 · ``)
All four are **pre-existing** (run4/earlier); this run created none. Tagged `pcnx-deleteafter=2026-07-21` (reaper-safe until then).

| Resource group | From | Session |
|----------------|------|---------|
| `rg-bya-notes-dev-4d29` | run4 App Service (patched again this run) | `4d294c61` |
| `rg-wetty-dev-42dd` | run4 Container Apps (fresh) | `42dd0b61` |
| `rg-wetty-dev-fafc` + `ME_cae-wetty-dev-fafc_rg-wetty-dev-fafc_eastus2` | prior Container Apps (resumed this run) | `fafc5cdc` |

**Cleanup — recommended before any future deploy run** (removes the contamination):


## Bottom line
- **3 real routing findings** (2 persistent, 1 flaky) — skill-side, unchanged from run4.
- **2 seeded-deploy failures are environmental** — caused by leftover run4 resources; clean up (or randomize RG names) and they'll deploy fresh again. Plus one genuine skill note: the resume-existing path uses a disallowed imperative `az appservice plan update`.
- Everything the earlier fixes targeted (DVWA, Kafka, Plan-Schema) passed.

## Artifacts
- Per-spec logs: `tests/results/overnight4-20260715-000148/<spec>.log`
- Per-spec results: `tests/results/overnight4-20260715-000148/<spec>/<ts>/results.jsonl` + `eval-results.md`
