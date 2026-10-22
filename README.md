# workflow-runner

Chain several single-purpose skills into one repeatable run. Define a workflow in YAML, say "run the monthly-business-review workflow for 2026-10", and the runner fires each step off as a subagent and drops the results in a versioned output folder.

Skills stay independent for one-off use. The workflow is just a thin orchestrator on top.

## Concepts

- **Workflow**: a YAML file in `workflows/<name>.yaml`.
- **Group**: runs in order. Groups execute one after another.
- **Step**: one subagent call. Steps in the same group run in parallel.
- **Prompt**: required on every step. Placeholders are replaced before dispatch.
- **Skill**: optional. If set, the subagent is told to use that skill. If omitted, the step is an ad hoc subagent call driven only by the prompt.

## Sample workflow

`workflows/monthly-business-review.yaml`

```yaml
name: monthly-business-review
params: [month]
groups:
  - id: gather
    steps:
      - id: delivery
        skill: pull-delivery-metrics
        prompt: Pull delivery metrics for {params.month}.
      - id: incidents
        skill: pull-incident-summary
        prompt: Summarize incidents for {params.month}, Sev1 and Sev2 only.
      - id: roadmap
        skill: pull-roadmap-status
        prompt: Pull roadmap status for {params.month}.

  - id: analyze
    steps:
      - id: trend-analysis
        skill: trend-analysis
        prompt: |
          Compare {params.month} against the prior 3 months.
          Delivery data: {steps.delivery.result}
          Incident data: {steps.incidents.result}
          Roadmap data: {steps.roadmap.result}
          Call out anything that moved more than 15%.

  - id: deliver
    steps:
      - id: mbr-deck
        skill: mbr-deck
        prompt: |
          Build the MBR deck for {params.month}.
          Analysis: {steps.trend-analysis.result}
          Raw data folders: {steps.delivery.dir}, {steps.incidents.dir}
      - id: exec-summary
        prompt: |
          Write a one-page executive summary for {params.month}.
          Use only: {steps.trend-analysis.result}
```

`gather` runs three steps in parallel. `analyze` waits for all three. `deliver` runs the deck and the summary in parallel (`exec-summary` has no skill, so it is a plain subagent call).

## Placeholders

| Placeholder | Resolves to |
|---|---|
| `{params.<name>}` | the param value for this run |
| `{steps.<id>.result}` | path to that step's `<id>-result.md` |
| `{steps.<id>.dir}` | path to that step's group folder (for generated json, pptx, html) |
| `{out}` | this step's own group folder (added automatically) |

Placeholders resolve to **paths**, and the subagent reads the files itself. A step can only reference steps in **earlier** groups. The runner checks this before anything runs.

## What a run looks like

```
.Output\monthly-business-review\2026-11-03-v1\
  _run.yaml          resolved params + copy of the workflow
  _summary.md        status table, rewritten at the end of every run
  gather\
    delivery-result.md
    incidents-result.md
    roadmap-result.md
  analyze\
    trend-analysis-result.md
  deliver\
    mbr-deck-result.md
    mbr.pptx
    exec-summary-result.md
```

- Folder is `.Output\<workflow-name>\<date>-v<n>`.
- Version is the highest existing version for that date plus 1. Deleting v1 while v2 exists gives you v3 next. Delete everything for the day and it goes back to v1.
- Each `<step>-result.md` is what the subagent returned, ending with a `## Files produced` list.
- A failed step writes `STATUS: failed` as the first line of its result file.

## Using it

Say any of these:

```
run the monthly-business-review workflow for 2026-10
dry run monthly-business-review for 2026-10
resume the last monthly-business-review run
resume monthly-business-review 2026-11-03-v1
```

- **Fresh run** is the default and always creates a new version folder.
- **Dry run** validates the workflow and prints each group, step, and fully resolved prompt. Nothing is created or dispatched. This is the fastest way to debug a workflow.
- **Resume** reuses the folder and its recorded params. Steps whose result file exists and isn't `STATUS: failed` are skipped. The rest rerun. The status table prints before anything runs.

## Failures

If a step fails, the other steps in its group finish, then the run stops. Fix the problem and resume.

Resume only reruns incomplete steps. To redo a step and everything downstream, delete that step's result file and the folders of the later groups, then resume.

## Tips

- Subagents don't share context. Give each skill what it needs through its prompt and the files it's pointed at.
- Keep step ids unique across the whole workflow.
- Parallel steps in one group write to separate result files, but they share the group folder. Have skills use distinct file names for generated artifacts.
- Start with a dry run whenever you change a workflow.
