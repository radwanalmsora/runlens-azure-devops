# RunLens: design notes

Version: 1.0.0. This document describes the behavior implemented in `src/core/analysis.ts` and `src/app.ts`. If the code and this document disagree, the code is the source of truth and this document needs fixing.

## 1. Goal

Answer two questions about a build, directly in its results tab:

1. **Why did it fail?** Name the first failed task, show its errors, and give a likely category.
2. **Why is it slower?** Compare the build and its tasks with recent builds of the same pipeline.

Every answer comes from a fixed rule that can be read in the source code. RunLens does not use machine learning, and it does not call any service outside Azure DevOps.

## 2. Non-goals (version 1.0)

- Flaky-test detection and a health score. These are planned as later modules.
- Azure DevOps Server (on-premises). Only `dev.azure.com` is allowed by the CSP.
- Pull-request or branch-level comparison. See section 7.
- Writing to any Azure DevOps resource.

## 3. Data sources

The tab runs inside the build results page and reads the build it is shown for.

| Call | Purpose |
| --- | --- |
| `GET /_apis/build/builds/{buildId}?api-version=7.1` | Build metadata, including `definition.id`. |
| `GET /_apis/build/builds/{buildId}/timeline?api-version=7.1` | Stages, jobs, and tasks, with result, start and finish times, and issues. |
| `GET /_apis/build/builds?definitions={id}&statusFilter=completed&queryOrder=finishTimeDescending&$top=6&api-version=7.1` | Recent completed builds of the same definition. |
| `GET /_apis/build/builds/{id}/timeline?api-version=7.1` (per baseline build) | Timelines for the baseline. |

- The base URL is `https://dev.azure.com/{organization}/{projectId}/_apis`. The organization comes from `VSS.getHost().name`, and the project ID comes from `VSS.getWebContext().project.id`.
- Requests use a Bearer token from `VSS.getAccessToken()`. RunLens never stores the token.
- The history request asks for six builds, removes the current build, and keeps at most five. A baseline timeline that fails to load is skipped, and the report still renders.

**Unverified in version 1.0:** the `queryOrder=finishTimeDescending` value and the payload shape of the `registerBuild` callback must be confirmed against a live organization before release.

## 4. Failure analysis

### 4.1 Choosing the failed task

1. Keep records with `type === "Task"` and `result === "failed"`.
2. Sort them by start time, earliest first.
3. The first record is the failed task. Stage and job failures are not used, because they usually repeat the task failure.

If no task failed, the report says `failed: false` and nothing else is shown.

### 4.2 Collecting errors

- Only issues with `type === "error"` and a non-empty message are used. Warnings are ignored.
- Whitespace is collapsed, and each message is cut to 300 characters.
- Duplicate messages are removed. At most 10 messages are kept.

### 4.3 Classification

The errors are joined with newlines into one text. The rules run in this order, and the first rule that matches decides the category. Matching is case-insensitive.

| Order | Category | Rule | Matches (summary) |
| --- | --- | --- | --- |
| 1 | environment | `disk-or-memory` | `ENOSPC`, `No space left on device`, `out of memory`, `OutOfMemory` |
| 2 | environment | `agent-or-network` | `lost communication with the agent`, `agent ... lost/offline`, `timed out`, `ETIMEDOUT`, `ECONNRESET`, `socket hang up`, `429`, `Too Many Requests` |
| 3 | dependency | `package-resolution` | `ERESOLVE`, `ETARGET`, `No matching distribution found`, `Could not resolve dependencies`, `Could not find a version that satisfies`, `Unable to find module`, `Cannot find module`, `Failed to resolve` |
| 4 | code | `compile-or-test` | `error TSnnnn`, `error CSnnnn`, `error[Ennn]`, `SyntaxError`, `AssertionError`, `FAILED`, `Tests failed`, `BUILD FAILED` |

If nothing matches, the category is `unknown`.

Why environment comes first: a full disk or a dropped agent connection often produces errors that look like code errors. Checking environment first avoids sending the developer to the source code for an infrastructure problem.

The report stores the name of the matched rule, so the user can see the exact reason for the category.

### 4.4 Suggestions

Each category has a fixed list of suggestions (2 to 3 items). The suggestions are generic next steps. They are not generated from the log.

## 5. Performance analysis

### 5.1 Definitions

- **Wall clock:** the earliest start time to the latest finish time across all timeline records. Records with missing or invalid timestamps are ignored in the calculation.
- **Task duration:** finish minus start for a task record. A task name that runs in several jobs is summed, so the figure is the total time spent on that task.
- **Share of wall clock:** a task's summed duration divided by the wall clock. Parallel jobs can make the total exceed 100%, and that is expected.

### 5.2 Slowest tasks

The current build's tasks are sorted by summed duration, and the top 10 are shown.

### 5.3 Baseline

- The baseline is the completed builds of the same definition, up to five, from section 3.
- The median is used rather than the mean, so one outlier does not move the comparison much.

### 5.4 Build regression

Reported only when there are at least **2** baseline wall-clock values. It is flagged when both conditions hold:

- the current wall clock is at least **20%** above the baseline median, and
- it is at least **60 seconds** above the baseline median.

### 5.5 Task regression

For each task in the current build:

- Need at least **2** baseline builds that contain that task name. Otherwise the task is not compared.
- Baseline value: median of that task's summed durations across those builds.
- Flagged when the task is at least **25%** slower and at least **30 seconds** slower than the baseline median.
- At most the top 5 regressions, sorted by absolute increase, are shown.

### 5.6 Why both a percentage and an absolute threshold

A percentage alone flags tiny tasks that jump from 1 second to 2 seconds. An absolute threshold alone hides real slowdowns in short tasks. Requiring both keeps the report focused on changes a developer would notice.

## 6. Presentation rules

- All build-provided text (task names, error messages) is rendered with `textContent`. It is never inserted as HTML.
- The page uses the Azure DevOps theme, so it follows the user's light or dark setting.
- The CSP limits network access to `dev.azure.com`.

## 7. Known limitations

These are known limits of the rules as implemented, not bugs to be fixed silently:

- **No branch filter.** The baseline takes the last five completed builds of the definition, from any branch. A build on a feature branch can be compared with a main-branch build, and the result can mislead.
- **Failed builds are included in the baseline.** `statusFilter=completed` includes failed and partially succeeded builds. A baseline that failed early can look artificially fast, which makes a normal build look like a regression.
- **Classification is a heuristic.** Unusual error text falls into `unknown` or into the wrong category. The matched rule and the raw errors are shown so the user can check.
- **Only the first failed task is analyzed.** If several tasks fail, the others are not described.
- **Pipeline changes are not detected.** Adding or removing steps changes the durations that are compared, and RunLens does not know why.
- **Task names are the matching key.** Two different tasks with the same display name are merged.

Planned improvements, in order of expected value: a branch filter defaulting to the build's source branch, excluding failed builds from the baseline, and showing the number of baseline builds used in the UI.

## 8. Testing

- Unit tests in `src/test/analysis.test.ts` cover the rules, the failed-task choice, error de-duplication and limits, the wall-clock calculation, and both regression thresholds.
- Analysis code has no browser or network dependencies, so tests use hand-written timelines.
- Live testing against a real organization has not been done yet and is required before the first public release.
