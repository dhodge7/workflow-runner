---
name: workflow-runner
description: Run a named multi-step workflow defined in YAML, where each step calls a skill (or an ad hoc prompt) as a subagent. Steps are organized into groups; groups run in order and steps inside a group run in parallel. Use this whenever the user says "run the <name> workflow", "kick off the monthly business review", "run workflow X for <params>", "resume the last run", "dry run workflow X", or wants several skills chained or fanned out together, even if they don't say "workflow". Do NOT use for a single one-off skill call.
---

# Workflow Runner

Runs a workflow file: ordered **groups** of **steps**. Each step is one subagent call. Steps in the same group run in parallel. Groups run one after another. Every step writes a `<step>-result.md` into a per-group folder inside a versioned run folder.

You are the orchestrator. Keep your own context small: pass file **paths** to subagents, never paste file contents into prompts, and do not read result files yourself unless you need to check status.

## Finding the workflow

Workflows live in the `workflows` subfolder of this skill (the folder next to this SKILL.md).

**If the user gives a path** (it contains a path separator, or ends in `.yaml` or `.yml`), use that file directly, in any folder. Resolve relative paths from the current working directory. If it doesn't exist, say so and stop.

**Otherwise treat what they said as a name:**
1. Normalize it: lowercase, and treat spaces, hyphens and underscores as the same character. Ignore filler words like "the", "workflow" and "run".
2. Normalize each file name in `workflows\` and each file's `name:` field the same way, and look for an exact match.
3. If there's no exact match, look for workflows whose normalized name contains the user's text. If exactly one does, use it and tell the user which one you picked.
4. If several match or none do, list the available workflows and ask which one.

If the user asks to list workflows, list the files in `workflows\` with their `name:` values.

The output folder always uses the `name:` field from the YAML, not the file name or what the user typed.

## Workflow file

Workflows live in `workflows/<name>.yaml` (relative to the project root; if the project keeps them elsewhere, use that location). The file name is the workflow name.

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
  - id: analyze
    steps:
      - id: trend-analysis
        skill: trend-analysis
        prompt: |
          Compare {params.month} against the prior 3 months.
          Delivery data: {steps.delivery.result}
          Incident data: {steps.incidents.result}
          Call out anything that moved more than 15%.
```

Fields:
- `name`: workflow name, used as the output subfolder.
- `params`: list of parameter names. If the user didn't supply a value, ask once before starting.
- `groups[].id`: group id, used as the group's folder name.
- `groups[].steps[].id`: step id. Must be unique across the whole workflow.
- `steps[].skill`: optional. Skill the subagent should use. If omitted, the step is an ad hoc subagent call driven only by `prompt`.
- `steps[].prompt`: the task-specific prompt. Required.

## Placeholders

Resolve these in each `prompt` before dispatch:

| Placeholder | Resolves to |
|---|---|
| `{params.<name>}` | the param value for this run |
| `{steps.<id>.result}` | full path to that step's `<id>-result.md` |
| `{steps.<id>.dir}` | full path to that step's group folder (for generated json/pptx/html) |
| `{out}` | this step's own group folder (added automatically; rarely written by hand) |

Placeholders resolve to **paths**. The subagent reads the files itself.

## Validate before running

Fail before anything executes if:
- a placeholder references a step id that doesn't exist
- a placeholder references a step in the **same** group or a **later** group (same-group steps run in parallel, so the file won't exist yet)
- a step id is duplicated
- a referenced `skill` isn't an available skill
- a param is used but not declared

Report every problem found, not just the first.

## Output folders

Base output folder is `.Output` (use the project's configured base if different).

```
.Output\<workflow-name>\<date>-v<n>\
  _run.yaml
  <group-id>\
    <step-id>-result.md
    <any files the step generated>
```

**Run folder creation:**
1. `<date>` is today as `YYYY-MM-DD`.
2. List existing folders in `.Output\<workflow-name>\` matching `<date>-v<n>`.
3. New version = (highest `n` for today's date) + 1, or `1` if none exist. Do not fill gaps: if v1 was deleted and v2 exists, the next run is v3.
4. Create with a plain non-recursive `mkdir` that fails if the folder exists. On failure, increment and retry.
5. Write `_run.yaml` into it: the resolved param values plus a copy of the workflow file as it was when the run started.

Create each group folder just before that group runs.

## Running

For each group, in order:

1. Create the group folder.
2. Resolve every step prompt in the group.
3. Dispatch **all steps in the group at the same time**, as parallel subagents in a single batch. A one-step group is just one subagent.
4. Wait for all of them to finish.
5. Check each step's result file (see Status below).
6. If every step succeeded, go to the next group. If any failed, stop (see Failures).

### Subagent prompt

Build each subagent's prompt in this order: the skill line (only if the step has a `skill`), then the resolved `prompt`, then the footer.

```
{if skill} Use the {skill} skill. {end}

{resolved prompt}

---
Write all files you generate to: {out}
When finished, write your final response to: {out}/{step-id}-result.md
End that file with a "## Files produced" section listing each file you created, one line each with a short description.
If you cannot complete the task, write the result file anyway with "STATUS: failed" as the first line followed by the reason.
```

### Status

A step is **complete** when `<step-id>-result.md` exists and its first line is not `STATUS: failed`.

If a subagent returns but did not write the result file, write what it returned into `<step-id>-result.md` yourself (append a `## Files produced` section from whatever it created in the folder). If the subagent errored out, write the result file with `STATUS: failed` and the error.

## Failures

Let the other steps in the current group finish, then stop. Do not start the next group. Report which steps failed and why, and tell the user they can fix the issue and rerun with `--resume`.

## Resume

"Resume" with no version means the latest version folder for that workflow. "Resume 2026-11-03-v2" means that exact folder.

Resume reuses the folder and its recorded params (don't ask for params again). Skip any step that is already complete. Rerun the rest, group by group, as normal. If the user asks for a fresh run, make a new version folder instead; this is the default whenever they don't say resume.

## Dry run

If the user asks for a dry run or a plan, validate and print the run: the folder that would be created, each group, each step, and each step's fully resolved prompt with its paths. Create nothing and dispatch nothing.

## When done

Reply with a short summary:
- the run folder path
- each group and step with status
- the key deliverables (generated pptx/html/json files) with paths

Do not repeat the contents of the result files. If the user wants to see one, point them to the path.

## Run summary

At the end of every run or resume, including when a run stops on failure, write `_summary.md` in the run folder, overwriting any previous one:

```
# <workflow-name> <date>-v<n>
Params: month=2026-10
Status: complete | stopped at group <id> | in progress

| Group | Step | Status | Result |
|---|---|---|---|
| gather | delivery | ok | gather\delivery-result.md |
| gather | incidents | failed: <reason> | gather\incidents-result.md |
| analyze | trend-analysis | not run | |
```

Status values: `ok`, `failed: <reason>`, `not run`, `skipped (already complete)` for steps a resume didn't rerun.

On resume, start by recomputing status from the result files and print this table before running anything, so the user sees what will be skipped and what will run.