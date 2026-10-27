# Before and after

Loaded from the Before and after section of `SKILL.md`. Steps 2 and 3 run once per side, before first, with the same numbered steps; only the URL and the evidence directory differ. Step 2 runs again on the after side because apps on `localhost` share cookies across ports, so logging into one can replace the other's session. Each side checks its endpoint targets against its own session origin. Step 4 runs once, after both sides.

The before side records observations, not verdicts. Base code is expected to differ at the step that shows the change, so before-side steps carry no PASS or FAIL. Only the after side fills the UI flow table and the VERDICT. The before side's numbers feed the measured sentence and, for an API change, the Changed behavior table. A before-side case that cannot run writes `Before: unavailable, <reason>` in its Before cell.

## Where the before side runs

The merge base is the one `done` recorded. When `browser-qa` runs on its own, I take the base from `gh pr view --json baseRefName` when a PR exists and otherwise select it as `done`'s section 1 does, then refresh it and take the merge base with that section's fetch and `merge-base` commands.

At the start of every run headed for a PR I delete `.qa/pr-evidence.md`. I take the first source that works, in this order, and name it in the report's Test Environment.

1. **An earlier capture of base code.** A before set taken earlier in this task, for example a bug repro run before any edit, counts only when its recorded commit is the merge base and its steps, account, seed data, and widths match this run. Otherwise I capture fresh.
2. **A base worktree.** This is the default. I check out the merge base outside the repository, so it never shows in `git status`, in a temporary directory (the session scratchpad when the agent has one):

   ```bash
   git worktree add --detach "<temp-dir>/qa-base" "<merge-base-sha>"
   ```

   I copy in the untracked env files the app needs to boot and install dependencies with the repo's package manager. Base servers must not change anything the head side uses. Before starting one I read its start command and config, and one that deploys code (a hosted dev backend such as `convex dev`) or migrates or syncs the schema at boot (an ORM `synchronize` or `sync({ alter })` setting) sends me to source 3. Otherwise I start a base copy of every server the flow calls, each on a free port, and point the copied env files at those ports.

   The steps I pass Step 3 include a network check after each API-triggering step that no before-side request reaches a head server's port, and I read the head servers' logs for requests during the before run, since a server-side call never reaches the browser. A request that reaches a head server stops the before side at once. When it may have written data I tell the user before the after side runs. Then, as when network or log evidence is missing, I record it, clean up, and move to source 3.

   Both sides read the same local database. For a flow that writes, the before side needs its own data, through a database clone the base env points at or a reset between sides. Both are database operations, so I ask the user once which to use. Without an answer the before side stops at the first step that writes, and each step it skipped gets `Before: unavailable, no separate data for a writing step`.

   On every exit from this source, success or failure, I stop the base servers and run `git worktree remove --force "<temp-dir>/qa-base"`, which also deletes the copied env files. When a base server fails to boot or the base app cannot render the flow, for example against a migration only the branch has, I record the exact error, clean up, and move to source 3.
3. **A deployed base environment.** A dev or staging URL running the base branch. I read the revision it reports and record it on the Before line. A flow that writes runs there only under a Step 3 mutation preflight `ready` card that includes restoration steps; without one I move to source 4. Its data differs from local seed data, and the fragment's data line says so.
4. **No source works.** The fragment says `Before: unavailable, <reason>` in place of the before half.

The Browser evidence rule in `done`'s readiness card decides whether a `Before: unavailable` line, or a Before revision other than the merge base, can publish.

A new screen has no before state to photograph. I capture the place a user would reach it from, for example the navigation without the new link, or write `Before: route did not exist` when there is no such place. A 404 screenshot shows nothing a reviewer needs.

## What both sides share

- The same numbered steps, account and role, seed records, and widths. A pair whose inputs differ compares nothing.
- Before a side is captured fresh I create its directory, `.qa/before/` or `.qa/after/`, and empty it, so a file from an earlier run can never stand in for one this run failed to write. A reused source 1 capture keeps its directory.
- Widths: 1920 always, the floor the Recording section sets, plus each width the change targets, plus 390 when responsive layout changed. At the step that shows the change I resize to each width, widest first, take one screenshot per width, and resize back to 1920 before the next step. Right before each shot I read `innerWidth` through the driver's evaluate hook. When it is not the width in the file name I resize and read once more, and when it is still wrong I take no shot. On the before side the pair's place says `Before: unavailable, the page would not resize to <width>px`; on the after side the missing shot fails its step. On `agent-browser` 0.31.1 a resize can report success and leave the page at its old size, which this read catches.
- Evidence goes to `.qa/before/` and `.qa/after/` in place of `.qa/`, each given to the Step 3 subagent as an absolute path under the head repository's root. Comparison shots are `<NN>-<step>-<width>.png` with the same `<NN>-<step>` on both sides, so each image has its pair. Step slugs use lowercase letters, digits, and hyphens only, because `--attach` reads a `#` as the start of alt text. Each side's recording is `flow` with the recorder's own extension, in its directory.
- One measured sentence per side, saying what a reviewer would check, with numbers: "rows are 77px instead of 49px", "the table is 1172px wide inside a 1036px card". I take the numbers from the driver's evaluate hook (`getBoundingClientRect`, `scrollWidth` against `clientWidth`, computed styles), never by eye.
- After-only shots for nearby surfaces the change must not break, one sentence each on what still holds, for example "Kanban still shows the close tick".
- The Step 3 subagent never reads this file. I write each resize, width read, network check, comparison shot, and measurement into the numbered steps I pass it, with the exact file name, so it runs them like any other step.

## Limits

- `gh` attaches at most 50 files per command. When the set would exceed that, I drop after-only shots first, then widths the change does not target.
- A video over 10 MB does not upload on a GitHub Free plan. I hold every recording to 10 MB unless the user confirms a paid plan, where the cap is 100 MB. I re-record only the steps that show the change, and when the file is still over, that side keeps its screenshots and says `Recording: over the upload cap` in place of the embed.
- Every screenshot publishes whatever the page shows. I run on synthetic or redacted seed data, and the fragment's data line names it, so `file-pr` can apply its data approval rule.

## The PR fragment

After Step 4 I write `.qa/pr-evidence.md`. `gh --attach` replaces a body reference with the uploaded file's URL when the reference resolves to the same absolute path as an `--attach` argument, so every path here is relative to the repository root, where `file-pr` runs `gh`. Only `![...](...)` embeds outside code get replaced, so the fragment names files through embeds and nowhere else. A video embed alone in its paragraph becomes a playing video. Anywhere else it degrades to a link.

```markdown
## Before and after

Recorded against <data source>. <One line on the records that matter, for example "One task has a long title and six assignees">.

**Before.** <measured sentence>

![Before, 1920px](.qa/before/03-task-list-1920.png)

![Before, 1440px](.qa/before/03-task-list-1440.png)

![Before recording](.qa/before/flow.webm)

**After.** <measured sentence>

![After, 1920px](.qa/after/03-task-list-1920.png)

![After, 1440px](.qa/after/03-task-list-1440.png)

![After recording](.qa/after/flow.webm)

<What still holds>:

![After, kanban 1920px](.qa/after/05-kanban-1920.png)

<the QA Test Results report from SKILL.md, in its PR form>
```

A recording embed appears only when that file exists. Otherwise its side says `Recording: unavailable, the browser could not record video`, and the driver's name stays in the chat report. An API change with no UI writes the same file without the Before and after section. Its before side is the Changed behavior table inside the report.
