---
name: live-progress
description: For long tasks (about 10+ minutes or 6+ steps), or when asked for progress or an ETA, publish a live progress page with steps, a progress bar, elapsed time and ETA, updated at each step.
---

# Live Progress

At the start of a long task, publish a page the user can open at any time to see what you are doing, each step's status, a segmented progress bar, elapsed time, time left and the expected finish time. The page recomputes timers and the ETA every second; you only write progress at checkpoints.

## When to use

- Tasks you expect to take about 10 minutes or more, or with 6 or more distinct steps.
- The user asks for a progress bar, progress updates, an ETA or "how long" (in any language, e.g. 进度条, 还要多久).
- Not when the user has said not to ("no progress page", "不要进度页") for the rest of that conversation, or has said to use it only when asked. Not for short tasks: each checkpoint costs one or two tool calls.

You need the `Artifact` and `ArtifactData` tools (`ArtifactData` is usually deferred: load it with ToolSearch `select:ArtifactData`). Without them (for example Claude Code in a terminal), track the steps with the task list (TaskCreate / TaskUpdate) or a short checklist in your replies instead.

## How it works

The page writes nothing. It subscribes to one document, `run/state`, in the artifact's database. You merge-update that document with `ArtifactData` at checkpoints, and anyone with the page open sees the change at once.

## 1. Plan

Split the task into 3–12 steps. Give each an `est`: realistic wall-clock minutes, including tool calls and writing. Starting points: a web search plus reading a few pages, 1–2 · reading a batch of files, 1–3 · writing about 1,000 words, 2–4 · writing code, running it and fixing one round, 3–8 · building and checking a file, 3–6 · a subagent research task, 5–15. Lean slightly high; the page recalibrates from real timings.

## 2. Read the clock

Run `date +%s` (Unix seconds). Every write carries timestamps. Reuse a reading less than a minute old, or append `; date +%s` to a Bash command you are running anyway.

## 3. Create and publish the page

The page is `template.html` in this skill's folder (the base directory shown when the skill loads). Copy it with a 2–4 word title filled in, such as "Q3 report progress":

```bash
python3 -c 'import sys;t=open(sys.argv[1],encoding="utf-8").read();open(sys.argv[3],"w",encoding="utf-8").write(t.replace("__TITLE__",sys.argv[2]))' "<SKILL_DIR>/template.html" "<page name>" progress-<slug>.html
```

If `template.html` is missing, do not rebuild the page from memory: tell the user in one line and use the task list instead.

Use a new file name for each task (the same path overwrites the previous page), in the working directory or scratchpad. Publish it with the Artifact tool: `file_path` = that file, `icon` = `"progress"`, `description` = one sentence about the task, `capabilities` = `{"db": {}}`. The capability is required; without it the page cannot read anything.

## 4. Save the state file

Right after publishing, write `progress-state.json` in the working directory:

```json
{"url": "<artifact URL>", "skill_dir": "<SKILL_DIR>", "title": "<task title>", "lang": "en"}
```

This is how you find the page again if your context is compacted in the middle of the task (step 7).

## 5. Write the initial state

`ArtifactData` `set`, `collection` `"run"`, `doc_id` `"state"`:

```json
{
  "title": "Clean the 2025 sales data and write the quarterly report",
  "lang": "en",
  "status": "running",
  "startedAt": 1790300000,
  "updatedAt": 1790300000,
  "activity": "Reading 12 CSV files",
  "steps": {
    "s01": {"n": 1, "title": "Read and check the data files", "est": 3, "status": "running", "startedAt": 1790300000, "done": 0, "total": 12, "unit": "files"},
    "s02": {"n": 2, "title": "Clean the data", "est": 8, "status": "pending"},
    "s03": {"n": 3, "title": "Compute quarterly metrics", "est": 5, "status": "pending"},
    "s04": {"n": 4, "title": "Write the report", "est": 10, "status": "pending"}
  },
  "log": {"1790300000": "Task started"}
}
```

- `steps` is an object keyed `s01`, `s02`… (not an array), so later writes can change one step. `n` sets the order; use 2.5 to insert a step.
- `lang` is `"zh"` or `"en"`, matching the user's language. Write titles, notes, activity and log entries in the user's language too.

If this first write is refused (the user declines the approval, or a permission error comes back), do not retry: tell the user in one line that live progress is off for this task, and continue with the task list. Otherwise tell the user in one line that the progress page is live and they can open the card at any time.

## 6. Update at checkpoints

One `ArtifactData` `update` per checkpoint, with only the fields that changed: nested objects merge and everything else stays. Pass the version from the previous write as `if_version`; on a version mismatch, `get` once and redo the write.

Checkpoints:

- A step ends and the next one starts: one write for both.
- A step works through many similar items (files, pages, sections): set `total`, then update `done` about every 10% or every 3–5 minutes, never per item.
- The plan changes: add steps under new keys; mark dropped ones `"status": "skipped"` with a `note`.
- Something goes wrong and you change approach: add a log entry with `"level": "warn"`.
- You stop to ask the user and wait: write `{"status": "waiting", "waitingSince": T, "updatedAt": T}`. When you resume at T2, write `{"status": "running", "waitingSince": {"__delete__": true}, "updatedAt": T2, "steps": {"<current key>": {"pausedSec": <its previous pausedSec, or 0> + (T2 - T)}}}`. Time spent waiting then does not count against the ETA.

Example, step 1 done and step 2 started:

```json
{
  "updatedAt": 1790300240,
  "activity": "Cleaning the data: merging duplicate customer records",
  "steps": {
    "s01": {"status": "done", "endedAt": 1790300240, "done": 12, "note": "2 files were GBK-encoded; converted"},
    "s02": {"status": "running", "startedAt": 1790300240}
  },
  "log": {"1790300240": "Files read; cleaning started"}
}
```

Write `activity` as one plain sentence about what you are doing now, without tool names. Log keys are timestamp strings; values are a sentence or `{"text": "...", "level": "warn"}` (`level`: `warn` or `error`). Log milestones only: start, step done, a problem, a change of plan; about 30 entries per task at most.

## 7. If you lose track

After context compaction or a long gap you may no longer see the page URL or the step plan. Before continuing the task, read `progress-state.json`, then `ArtifactData` `get` `run/state`: it holds every step, its status and the current version. Carry on from there.

## 8. Finish

Always write the final update, even when the user has already seen the result in the conversation. Otherwise the page keeps showing "running" and later "may have stopped".

```json
{"status": "done", "endedAt": 1790302100, "updatedAt": 1790302100, "activity": "", "summary": "Report done: 18 pages with 6 charts. Q3 revenue up 12% on Q2.", "steps": {"s04": {"status": "done", "endedAt": 1790302100}}, "log": {"1790302100": "Task finished"}}
```

On failure write `"status": "failed"`, set the current step to `"failed"`, and say in `activity` where it stopped and what the user needs to do.

If you have a push-notification tool (for example `PushNotification`), also send one short notification when the task finishes or fails, so a user who walked away knows.

## Fields

`run/state`: `title`, `lang`, `status` (`running` | `waiting` | `done` | `failed`), `startedAt` / `updatedAt` / `endedAt` (Unix seconds; milliseconds and ISO strings also work), `waitingSince`, `activity`, `summary`, `steps.<key>`, `log.<timestamp>`.

Each step: `n`, `title`, `est` (minutes), `status` (`pending` | `running` | `done` | `failed` | `skipped`), `startedAt` / `endedAt`, `pausedSec`, `done` / `total` / `unit` (item progress such as 7 / 12 files), `note`.

## How the page estimates time

- Pace k = (actual minutes of finished steps + evidence from the running step + a prior) ÷ (their estimates + the prior), kept between 0.3 and 5. Waiting time is excluded.
- Time left = what remains of the running step + the remaining steps' `est` × k. A running step with `done` / `total` uses its measured rate per item; an overdue step counts a quarter of its elapsed time as remaining.
- Each bar segment is sized by its step's `est`.
- No update for 1.5 × the running step's `est` (at least 6 minutes) shows a warning; three times that shows "may have stopped".

So write realistic `est` values, and give long loops a `total` with regular `done` updates.
