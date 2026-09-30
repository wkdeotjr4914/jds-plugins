# workflow-view.md - the live workflow viewer (force-load)

go keeps a live picture of the run that the user can open in a browser at any time: a React Flow
diagram of the run's phases and the plan's tasks, colored by status, refreshed every few seconds.
It is a **window onto the run, not part of it**: nothing in go reads it back, no gate depends on
it, and a failure to write it never stops or slows the run.

## Files

| File | Written | By |
|---|---|---|
| `.flow-maker/workflow.html` | once per run, copied verbatim from this skill's `assets/workflow.html` | orchestrator |
| `.flow-maker/workflow-state.js` | rewritten whole at every update point below | orchestrator |

Both live inside the gitignored `.flow-maker/`, so they are never committed. The HTML is static -
never edit or regenerate it; all change goes through the state file. The page reloads the state
file with a `<script>` tag, so it works when opened straight from disk (`file://`), with no server.
It loads React and React Flow from a CDN; offline it falls back to a plain list.

## State file format

The file is one JavaScript assignment of a JSON object - nothing else in the file:

```js
window.__FLOW_MAKER_WORKFLOW__ = {
  "rev": 7,
  "updated": "2026-09-29T10:12:03Z",
  "title": "<the feature, in the user's language>",
  "status": "running",
  "now": "<one plain sentence: what is happening right now>",
  "labels": { "done": "<'Done' in the user's language>", "...": "..." },
  "phases": [
    { "id": "prepare", "title": "<user language>", "status": "done" },
    { "id": "build", "title": "<user language>", "status": "running", "note": "<optional>" }
  ],
  "tasks": [
    { "id": "T1", "title": "<user language>", "wave": 1, "depends": [], "status": "done",
      "note": "<optional, one line>", "detail": "<optional, a few sentences>",
      "checks": [ { "cmd": "<command>", "exit": 0 } ] }
  ],
  "checks": [ { "cmd": "<command>", "exit": 0 } ],
  "log": [ { "t": "2026-09-29T10:12:03Z", "msg": "<one plain sentence>" } ]
};
```

- **`rev`** - an integer that goes up by one on every write. The page redraws only when it changes.
- **`updated`** - UTC ISO-8601 time of the write. The page warns when it is more than 10 minutes old
  while the run is not finished.
- **`status`** (the whole run) - `running` | `waiting` (paused on the user) | `blocked` (stuck, the
  user has been told) | `done` | `failed`.
- **`phases[].id`** - a fixed set, in this order; include every one from the first write, `pending`
  until reached:
  `prepare` (read the 3-doc, preconditions) - `validate` (graph validation) - `base` (branch or main
  set-up) - `scaffold` - `build` (all task waves) - `verify` (completion gate) - `docs` (harness
  create / update) - `review` (final review) - `approval` (user gate) - `archive` - `finish`
  (integration choice). A phase that does not apply to this run (e.g. no scaffold) is `skipped`.
- **`phases[].status`, `tasks[].status`** - `pending` | `running` | `review` | `waiting` | `done` |
  `concerns` (done with a flagged concern) | `blocked` | `failed` | `skipped`. Use exactly these.
- **`tasks[]`** - one entry per task in the Execution Graph, filled at the `validate` update.
  `id` and `depends` copy the graph exactly (they draw the edges; the page never shows ids).
  `wave` is the 1-based wave number go derived (the page lays out one column per wave).
- **`checks`** - verification commands that actually ran, with their captured exit codes: per task
  on the task, integration and completion gates at the top level. Record only commands that ran;
  never a check that did not run, never an exit code you did not capture.
- **`log`** - append one plain sentence per update, keep the last 30 entries.
- **`labels`** - the page's own words (status names, headings). The page ships English defaults; when
  the user's language is not English, write every key below once, in the user's language, and keep
  them in every later write:
  `pending running review waiting done concerns blocked failed skipped run_running run_waiting
  run_blocked run_done run_failed step parallel tasks checks activity details depends note
  select_hint no_state stale updated`.

## Update points

Write the state file at each of these moments, and at no others (it is a snapshot, not a trace):

1. **Start** - right after loading the 3-doc: copy the HTML, write the first state (all phases
   present, `prepare` running, `tasks` empty, `labels` filled).
2. **Graph validated** - fill `tasks` (all `pending`, with `wave` and `depends`); `validate` done or
   failed. On failure, run `status` = `blocked` and `now` says what the user must fix.
3. **Each phase change** - the phase that ends and the one that starts.
4. **Each task change** - started or dispatched (`running`), under a mid-run spec-review (`review`),
   landed (`done` / `concerns`, with its checks), `blocked` or `failed`. A parallel wave may share one
   write for the tasks dispatched together.
5. **Each gate** - integration gate and completion gate results into `checks`.
6. **Each time the run pauses on the user** - run `status` = `waiting` (or `blocked`), `now` states
   what is needed, in plain words; back to `running` when the user answers.
7. **End** - after `finish`: run `status` = `done`, every phase settled. Then move both files into
   the archive directory created at step 13 (`.flow-maker/NNN/`), so `.flow-maker/` stays clean.

## Rules

- **Only the orchestrator writes it.** Subagents never see or touch these files (they work in other
  worktrees and could race). Fold their returned summaries into the next write.
- **Rewrite the whole file** each time; never patch it in place. It must stay valid JavaScript whose
  right-hand side is valid JSON - a malformed write blanks the picture until the next one.
- **Silent.** Writing the state file is plumbing: never announce it, never narrate it. The only
  user-facing mention is the one-line pointer described in SKILL.md.
- **User's language, plain words.** `title`, `now`, `note`, `detail`, `log`, phase and task titles,
  and `labels` are user-facing: write them natively in the user's language, with no internal tokens
  (no task ids, wave numbers, risk labels, or flow-maker terms - the page adds "step 1, step 2" itself).
- **Truthful.** The picture reports the same evidence the gates use: a task is `done` only after it
  landed on the base and its checks ran; a check shows only a captured exit code.
- **Cheap.** Keep `detail` to a few sentences and `log` to 30 entries; the file should stay small.
- **Never load-bearing.** If a write fails, carry on and try again at the next update point.
