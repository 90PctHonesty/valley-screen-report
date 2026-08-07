# GitHub Pages deploy failures — incident notes (NOT SUBMITTED)

**Status: NOT filed with GitHub Support. See "Why this was not filed" below.**
**Last refreshed: 2026-08-06 19:47 UTC.**

Repository: `90PctHonesty/valley-screen-report` (public)
Site: https://90pcthonesty.github.io/valley-screen-report/
Account: 90PctHonesty (cruz.jaydan02@gmail.com) — Personal, Standard plan
All timestamps UTC.

---

## Why this was not filed

Three blockers, any one of which is sufficient:

1. **There is an active, declared GitHub incident covering exactly these two
   services.** githubstatus.com shows Actions and Pages both **red / Incident**,
   opened 15:22 UTC and still ongoing as of the 19:43 UTC update. GitHub
   engineers are actively engaged. A ticket would be redundant.
2. **The support form offers no applicable category.** For this account the
   contact flow lists only: Copilot, Codespaces, Repositories, Education,
   Sign-in issues, Billing and payments. There is no Actions or Pages option.
3. **Technical support is not included on this plan.** The account selector
   labels `90PctHonesty` as *Personal / Standard — "Technical support not
   included."*

**Correction to the earlier draft:** it stated that githubstatus.com showed no
incident and listed "GitHub incident" as ruled out. That was accurate when
checked at ~14:50 UTC, but is **no longer true**. The incident was declared at
15:22 UTC. That claim has been removed.

---

## GitHub's incident timeline (from githubstatus.com)

| Time | Update |
|---|---|
| 15:22 | Investigating reports of degraded performance for Actions |
| 15:41 | Actions experiencing degraded availability |
| 15:45 | "Some workflow runs are failing to start or failing partway through"; Actions REST API returning errors |
| 15:53 | Pages experiencing degraded performance |
| 16:19 | Pages operating normally (brief) |
| 16:27 | Pages degraded again; Actions still degraded |
| 16:33 | "Actions and Pages are experiencing degraded availability" |
| 17:02 / 17:40 / 18:11 | Mitigations applied and rolled out; runs still failing or delayed, queued jobs may time out |
| 18:46 | "Recovery is taking longer than we expected" |
| 19:43 | "Capacity remains constrained and jobs may still be delayed or fail while it recovers gradually" |

GitHub's own wording maps directly onto both observed faults:
"workflow runs are failing or delayed in starting, and some queued jobs may
time out."

**One discrepancy worth noting:** the first observed failure (run #4 attempt 1,
13:13–13:24 UTC) and its wedged re-run (13:48 UTC) both predate the 15:22 UTC
declaration by ~1.5–2 hours. Run #5's failure at 15:02–15:13 also slightly
predates it. So the underlying degradation appears to have started earlier than
GitHub's public incident window.

---

## Observed faults

### Fault A — Pages deployment accepted but never advanced

Runs that actually executed created a Pages deployment, then polled status ~114
times over 10 minutes receiving `deployment_in_progress` every time, and aborted.

Deploy job for run #5 (`31113923304`, job `92658559470`), 15:03:02 → 15:13:08:

```
Run actions/deploy-pages@v5
Fetching artifact metadata for "github-pages" in this workflow run
Found 1 artifact(s)
Creating Pages deployment with payload:
{
  "artifact_id": 8972856675,
  "pages_build_version": "f2053ffb5e7164da4b901102791ff5feaccbb3de",
  "oidc_token": "***"
}
Created deployment for f2053ffb5e7164da4b901102791ff5feaccbb3de, ID: f2053ffb5e7164da4b901102791ff5feaccbb3de
Getting Pages deployment status...
Current status: deployment_in_progress
      ... this pair repeats ~114 times over 10 minutes ...
Error: Timeout reached, aborting!
Error: Timeout reached, aborting!
Canceling Pages deployment...
Canceled deployment with ID f2053ffb5e7164da4b901102791ff5feaccbb3de
```

Run #4 attempt 1 produced an identical signature, deploy duration 10m 6s.

**Successful run for comparison** — run #3 (`31078451094`, job `92541568909`),
06:47:01 → 06:47:09:

```
Created deployment for 02d309f6caae2601ba22e8b81d638f68e09fdade, ID: 02d309f6caae2601ba22e8b81d638f68e09fdade
Getting Pages deployment status...
Reported success!
```

Same action version, same payload shape, same client behaviour. Run #3 returned
success on **poll #1** in 6 seconds. Runs #4 and #5 returned
`deployment_in_progress` on all **~114** polls.

### Fault B — re-runs never scheduled, and cannot be cancelled

| Run | Re-run triggered | `updated_at` | Jobs | Status at 19:45 |
|---|---|---|---|---|
| #4 `31104941358` | 13:48:28 | frozen at 13:48:28 | 0 | `queued` (5h 57m) |
| #5 `31113923304` | 16:32:10 | frozen at 16:32:10 | 0 | `queued` (3h 13m) |

`updated_at` never advances past the trigger instant; `GET .../jobs` returns
`total_count: 0`. Neither has been reaped. No deployment record was ever created
for either re-run — they never executed.

By contrast both **push-triggered** runs had runners assigned within ~3–5
seconds, so this is not uniform queue starvation.

**Cancellation attempts (4, all failed).** The Actions UI Cancel button issues:

```
POST https://github.com/90PctHonesty/valley-screen-report/suites/84373906169/cancel
     _method=put
     authenticity_token=<valid, from page form>
```

Once via the UI button, three times replayed from the browser console with the
page's own CSRF token and authenticated session cookies. Every attempt returned
**HTTP 302 → `/actions`** with the flash:

```
Failed to cancel workflow.
Sorry, something went wrong.
```

Run state and `updated_at` unchanged after every attempt. The request is
well-formed, authenticated and CSRF-valid; the refusal is server-side.

---

## Full run timeline (UTC, 2026-08-06)

| Time | Run | Attempt | Result |
|---|---|---|---|
| 06:21:02 | #1 `31076986613` | 1 | success, 27s |
| 06:35:56 | #2 `31077823840` (`ed9c678`) | 1 | success, 42s |
| 06:46:47 | #3 `31078451094` (`02d309f`) | 1 | **success**, deploy 8s — last good deploy |
| 13:13:50 | #4 `31104941358` (`acdcd10`) | 1 | **failure** — Fault A, deploy 10m 6s |
| 13:48:28 | #4 `31104941358` | 2 (re-run) | **wedged** — Fault B, still queued |
| 15:02:47 | #5 `31113923304` (`f2053ff`) | 1 | **failure** — Fault A, deploy 10m 6s |
| 16:32:10 | #5 `31113923304` | 2 (re-run) | **wedged** — Fault B, still queued |

Deployment records (`github-pages` environment):

| Deployment ID | Created | Final state |
|---|---|---|
| 5774910027 | 06:46:58 | success 06:47:10 — last good |
| 5780067764 | 13:14:00 | failure 13:24:11 (waiting → queued → in_progress → failure) |
| 5781920082 | 15:02:58 | failure 15:13:09 (waiting → queued → in_progress → failure) |

All deployment status records have empty description fields.

---

## Configuration

| Item | Value |
|---|---|
| Pages source | **Deploy from a branch** (legacy), branch `main`, folder `/ (root)` |
| Workflow | `pages-build-deployment` (GitHub-managed), `workflow_id` 328378634, `event: dynamic` |
| Deploy action | `actions/deploy-pages@v5` |
| Environment | `github-pages` |
| Custom domain | none |
| Enforce HTTPS | enabled (locked — default domain) |
| Repo files | `.nojekyll`, `README.md`, `index.html` |
| Pages settings page | no error or warning banner |

---

## Ruled out locally

- **Artifact size / malformation** — run #3 = 8,608 B, run #4 = 8,931 B,
  run #5 = 8,933 B. Single `github-pages` artifact each, all `expired: false`.
  Build job succeeded in 4–6s every time.
- **Repository content** — the only change between last success (`02d309f`) and
  first failure (`acdcd10`) is one commit touching only `index.html`, +19/−9
  lines: a date stamp (`Checked Wed Aug 5` → `Thu Aug 6`), the meta description,
  one body-text edit, one added listing block. No new files, no structural
  change, no size jump.
- **Pages configuration** — unchanged, no error banner.
- **Client-side cancel problem** — disproved by replaying the exact request.

**NOT ruled out — and now the leading explanation:** the ongoing GitHub
Actions/Pages incident.

## Not obtained

`GET /repos/90PctHonesty/valley-screen-report/pages` and `.../pages/builds`
return **404** unauthenticated (these require a token; api.github.com does not
accept browser session cookies). To read the Pages build record's `error` field:

```
gh api /repos/90PctHonesty/valley-screen-report/pages/builds/latest
```

---

## Recommended action

Wait for the incident to clear, then push once and watch. Do **not** use
"Re-run jobs" while the incident is active — both re-run attempts produced
permanently wedged, uncancellable queued runs. The two wedged runs
(`31104941358`, `31113923304`) should be left alone; GitHub will reap them.

Current impact: the site still serves the 06:47 UTC build. Content from
2026-08-06 has not published.
