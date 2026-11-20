---
title: How friction works
---

# How friction works

This is the long version of the "How it works" section in the
[friction README](https://github.com/martinwheeler/friction#how-it-works): what each part does,
when it runs, what it stores and why. It describes the code at commit
[`3e8ebab`](https://github.com/martinwheeler/friction/tree/3e8ebabe23260f3fa8009c8d332444a1810c1615).

## Diagrams

Each diagram is an interactive page with light and dark themes, zoom and export. Every box links
to the source lines it describes.

| Diagram | Shows |
|---|---|
| [System overview](diagrams/architecture.html) | every component, what it reads and writes, what leaves the machine |
| [One occurrence, end to end](diagrams/occurrence-sequence.html) | a failing tool call, the agent logging it, the next session hearing about it |
| [Item lifecycle](diagrams/item-lifecycle.html) | open, resolved, wontfix, merged, and how regressions reopen an item |
| [Nightly backfill](diagrams/backfill-workflow.html) | how past transcripts are mined for friction nobody logged |
| [Jira sync, one run](diagrams/jira-sync-sequence.html) | the pull and push steps when tracker sync is on |
| [Database schema](diagrams/schema-erd.html) | every table and column in `friction.db` |

The diagram sources are JSON files in
[`diagrams/src`](https://github.com/martinwheeler/friction-docs/tree/main/diagrams/src); see
[how the diagrams are built](diagrams/) to change or rebuild them.

## The idea

Agents lose time to things that are not the task: an expired token, a shell quoting rule, a port
held by a stale container. Every new session rediscovers the same problem. friction keeps one
shared list of these problems on the machine, so the second session to hit one already has the
workaround.

Three things make that work:

1. **Hooks record the facts.** Every tool call is timed and marked as passed or failed, without
   the agent doing anything.
2. **Agents supply the judgement.** Only the agent knows whether three failed commands were a
   real obstacle or ordinary iteration, so the agent names the problem with `friction log` or
   upvotes a known one with `friction bump`. The hook data then measures how much time it cost.
3. **The list comes back.** Every new session starts with the current worst items and their
   workarounds.

A nightly backfill catches what agents forgot to log, and an optional Jira sync shares the list
with a team.

## The parts

| File | Runs when | Does |
|---|---|---|
| `friction_hook.py` | on every hook event from Claude Code or Codex | records tool calls, prints the hot list at session start, nudges at the end of a turn |
| `friction.py` | an agent or you run `friction ...` | the CLI and the `Friction` class that owns the database |
| `backfill.py` | `friction backfill`, nightly at 03:30 by default | rebuilds tool calls from transcripts and asks a small model which failures were friction |
| `web.py`, `web.html` | always, as a background service | read-only heatmap on `http://127.0.0.1:8798` |
| `tracker.py` | `friction sync`, started in the background after changes | optional two-way sync with Jira Cloud |
| `SKILL.md` | loaded by agents | tells agents when and how to log |
| `installer.py`, `install.sh` | install, update, uninstall | links, hooks, services |

Everything is Python standard library. The only external program used is the `claude` CLI, and
only by the backfill.

## Where things live

| Path | What |
|---|---|
| `~/.local/share/friction/friction.db` | the database (SQLite, WAL mode). `FRICTION_DIR` moves it |
| `~/.local/share/friction/app` | the code, when installed with `curl ... \| bash` |
| `~/.local/share/friction/analyser` | empty working directory for the backfill's `claude -p` runs |
| `~/.local/share/friction/sync.lock` | held while a tracker sync runs |
| `~/.local/bin/friction` | symlink to `friction.py` |
| `~/.agents/skills/friction/SKILL.md` (and `~/.claude/skills`, `~/.codex/skills`) | symlinks to `SKILL.md` |
| `~/.claude/settings.json`, `~/.codex/hooks.json` | hook entries pointing at `friction_hook.py` |
| `~/.config/systemd/user/friction-*` | Linux services: `friction-web.service`, `friction-backfill.service` and `.timer` |
| `~/Library/LaunchAgents/io.github.martinwheeler.friction.*.plist` | the same services on macOS, logs in `~/Library/Logs/friction-*.log` |
| `~/.config/friction/config.ini` | tracker settings, only if you set up sync. `FRICTION_CONFIG` moves it |

## 1. Hooks record every tool call

The installer registers one command, `friction_hook.py`, for seven hook events. Claude Code sends
all seven; Codex sends six (it has no `PostToolUseFailure`) and gets `--agent codex` on the
command line. Each hook has a 5 second timeout and the script exits 0 on any error, so a broken
database can never break an agent session. `FRICTION_DISABLE=1` turns it off for a process and
its children.

Every event carries a `session_id` (Codex: `thread_id`). The hook upserts a `sessions` row with
the agent, working directory, project and transcript path, then:

| Event | What the hook does |
|---|---|
| `PreToolUse` | inserts an `events` row with `ok = NULL` (in flight), the tool name, its `tool_use_id` and a one-line summary of the input (the command for shell tools, otherwise key fields such as `file_path` or `url`) |
| `PostToolUse` / `PostToolUseFailure` | finds the open event by `tool_use_id` (falling back to the newest open event for that tool) and closes it with `ok`, duration and error text. If there is no open event, because the hook was installed mid-session, it records the call untimed |
| `SessionStart` | returns `additionalContext`: a paragraph saying the tool exists, then the hot list. Also starts a background tracker sync, at most every 10 minutes |
| `Stop` | asks the agent once whether its recent failures were friction (see [Nudges](#4-the-list-comes-back)). If it does not block, the turn is over: opens a `prompt` wait |
| `UserPromptSubmit` | closes the open wait: the user has answered |
| `PermissionRequest` | opens a `permission` wait, closed when the call finishes (or by the next call or prompt, if it was denied) |

Waits are `events` rows with `kind = 'wait'` and the reason in `tool`. A question tool
(`AskUserQuestion`, `ExitPlanMode`, Codex `request_user_input`) also records its whole run as a
wait when it finishes.

Subagents report with an `agent_id`. Their tool calls are recorded, but they get no SessionStart
context and no Stop nudge.

**Deciding that a call failed.** `PostToolUseFailure` always counts as failed. For
`PostToolUse`, the response is checked in this order: `is_error`, `success: false` or a non-zero
`exit_code` in a structured response; otherwise the first 300 characters of the text are checked
for a leading `error`, `exit code N`, `command failed`, `permission denied`, or an exit code
pattern with a non-zero value.

**Privacy on the way in.** Summaries and errors go through `redact()` before they are stored: it
replaces API keys (`sk-`, `ghp_`, `github_pat_`, `xox?-`, `AKIA`, `AIza`), JWTs, bearer and basic
credentials, `password=`-style assignments and passwords in URLs, collapses whitespace and cuts
the text to 500 characters. Raw events are deleted after 30 days (`friction prune`, also run at
the end of every backfill).

## 2. Agents name the friction

The agent, guided by `SKILL.md`, decides what counts. The rule of thumb: time lost to the
environment, tooling or instructions, not to the task.

### `friction log` refuses to make duplicates

`friction log "<title>" --detail "..." [--workaround "..."]` first searches existing items with
the title and detail. The search is SQLite FTS5 with the Porter stemmer over title, detail,
workaround and aliases, ranked by BM25 with weights 8, 2, 1 and 6, after dropping stop words. If
anything matches, it prints the matches with their workarounds and exits with code 3 **without
logging anything**. The agent then either runs `friction bump <id>` (same root cause) or re-runs
the same command with `--new` (genuinely different).

This is what keeps the list short. Without it, every agent would log its own phrasing of
"gh token expired".

### `friction bump` is the upvote

`bump` and a successful `log` both record an **occurrence**: one row saying this item cost this
session this much time. To fill it in, the CLI needs to know which session is calling:

1. `--session`, if given.
2. `FRICTION_SESSION`, `CLAUDE_CODE_SESSION_ID`, `CODEX_SESSION_ID` or `CODEX_THREAD_ID` from the
   environment.
3. Otherwise, the session with an in-flight tool call from the last two minutes whose summary
   contains `friction`. That call is the `friction bump` command itself, recorded by the
   `PreToolUse` hook a moment earlier.

The session row supplies the agent and project. The project is the first directory under
`~/code`, or else the first directory under your home (`~/code/web-app/src` is `web-app`).

## 3. Measuring the time it cost

An occurrence's wasted time comes from the recorded events wherever possible, not from the
agent's guess.

**The window.** By default the window starts at the beginning of the current failure streak:
take the unlogged failed calls in this session (those after the session's `mark_event`), start
from the newest, and walk back while consecutive failures are less than 10 minutes apart. The
window ends now. `--from <event id>` (ids from `friction timeline`) starts it somewhere else, for
friction that began earlier or involved no failed call.

**Active time only.** The window is not simply end minus start. Walking the calls in order, each
gap to the next call counts in full only if it is no longer than the call's own duration plus 5
minutes. A longer gap is the session sitting idle (you were away), so only the call's own
duration counts.

**Never the user's time.** Any part of the window covered by a wait is then taken out, however
short: from the end of a turn to the next prompt, a permission prompt, a question tool. That time
is pending user input, not friction. A wait whose closing hook never fired runs until the next
tool call. `friction retime` applies this to occurrences logged earlier: it reads each session's
waits from its transcript (as backfill does) and re-measures the occurrence over its original
window, which needs the raw events to still be within the 30 day retention.

**After logging,** the session's `mark_event` moves to the newest event, so the same failures
are not counted twice and the Stop nudge stops asking about them.

Each occurrence records where its number came from, and reports keep these apart:

| `timing` | Source |
|---|---|
| `hook` | measured from live hook events |
| `transcript` | measured from events the backfill rebuilt from a transcript |
| `estimate` | the agent's `--wasted 5m`, used only when the session has no events at all. Shown as `~est` and "estimated" |
| `none` | no events and no estimate |

## 4. The list comes back

**The hot list.** `hot()` takes open items with at least one occurrence in the last 14 days and
ranks them by `hits × 60 + seconds wasted`, so an item hit often but quickly still ranks near one
that cost a lot once. The top 5, with workarounds, go into every new session through the
`SessionStart` hook (on startup, resume, `/clear` and compaction). `friction hot` shows the same
list.

**The Stop nudge.** When an agent finishes a turn with two or more failed calls that are neither
logged nor already asked about, the Stop hook blocks once with a short message: if that was
friction, log it; if it was normal work, ignore this. It then records the newest event as
nudged, so it asks once per streak. It never fires when Claude Code is already continuing
because of a Stop hook (`stop_hook_active`).

## 5. Item lifecycle

See the [lifecycle diagram](diagrams/item-lifecycle.html).

| Status | Meaning | Gets there by |
|---|---|---|
| `open` | a live problem, eligible for the hot list | being created; `reopen`; a regression |
| `resolved` | the cause is fixed; the fix note is kept | `friction resolve <id> --fix "..."`, or the Jira issue moving to done |
| `wontfix` | accepted, stop showing it | `friction wontfix <id> --reason "..."` |
| `merged` | a duplicate folded into another item | `friction merge <src> <dst>` |

- **Regression.** A new live occurrence on a `resolved` item, timed after `resolved_at`, reopens
  it and increments `regressions`. The output says `REGRESSION` and repeats the old fix note.
  Backfilled occurrences from before the fix do not count. Hits on a `wontfix` item do not reopen
  it.
- **Merge.** Occurrences move to the destination, the source title becomes a searchable alias,
  workarounds are combined, and the destination is open if either was. Any lookup of the old id
  follows `merged_into` to the survivor.

`friction workaround <id> "..."` sets or (`--append`) adds to the workaround at any point.

## 6. Backfill: friction nobody logged

See the [backfill diagram](diagrams/backfill-workflow.html). `friction backfill --since 2d` runs
every night at 03:30 as a systemd timer or LaunchAgent. It has two stages so the model only sees
sessions that might contain friction.

**Stage 1, plain Python.**

1. Find transcripts modified within `--since` and idle for at least 15 minutes:
   `~/.claude/projects/*/*.jsonl` and `~/.codex/sessions/*/*/*/*.jsonl`. Skip any already
   processed at the same modification time (tracked in `backfill_sessions`), the current session,
   and the analyser's own working directory.
2. Parse each one into timed tool calls. Claude Code transcripts pair `tool_use` with
   `tool_result` blocks; Codex rollouts pair `function_call` with `function_call_output`, using
   the same failure rules as the hook. Waits come out of the transcript too: in Claude Code, the
   gap from the agent's last message to each prompt the user typed; in Codex, from
   `task_complete` (or `turn_aborted`) to the next `task_started`; and the run of question tools.
   Permission prompts are not visible in transcripts, so their time stays inside the call.
3. Import the calls as `events` with `source = 'transcript'`, unless the live hook already
   recorded that session.
4. Keep the session only if it has at least `--min-failures` (2) unlogged failures. Otherwise mark
   it `clean` and move on.

`--dry-run` stops here and lists the candidate sessions.

**Stage 2, the analyser.** For up to `--limit` (40) sessions per run:

1. Build an excerpt: each unlogged failure with two calls before and three after, gaps elided,
   along with the list of existing items and anything already logged for that session.
2. Run `claude -p --model haiku` with no tools, no MCP servers, no settings files, no session
   persistence, `FRICTION_DISABLE=1` and a fixed system prompt that defines friction. It must
   answer with a JSON array of findings: an existing `item_id` or a new title, detail, workaround,
   and the event ids where the friction began and ended.
3. File each finding through the same `occur()` path as a live bump, with `source = 'backfill'`.
   Findings that cite event ids not in the excerpt are dropped. The session's mark then moves past
   everything analysed.

If the analyser fails, the session is left unprocessed and the next run tries again. Backfill is
the only part of friction that sends anything to a model; skip the timer (`--no-services`) if you
do not want that.

## 7. The web view

`web.py` is a `ThreadingHTTPServer` on `127.0.0.1:8798` that never writes to the database:

| Route | Returns |
|---|---|
| `/` | `web.html`, a single page with no external assets |
| `/api/report?days=14&project=&agent=` | the same data as `friction report --json`: totals, per-item rows with a per-day breakdown, and project and agent facets |
| `/api/item?id=N` | one item, its stats, and up to 100 occurrences with the recorded events behind each |

## 8. Jira sync (optional, off by default)

See the [sync diagram](diagrams/jira-sync-sequence.html).

**Switching it on.** `friction sync --enable` writes a template to
`~/.config/friction/config.ini` the first time. Once filled in, running it again checks the
connection (`/myself` and the project) and only then sets `enabled = true`. `--disable` sets it
back to `false`. With sync off, `kick_sync()` returns immediately and nothing touches the
network.

**When it runs.** After any command that changes data (`log`, `bump`, `workaround`, `resolve`,
`wontfix`, `reopen`, `merge`, `backfill`), the CLI starts `friction sync --quiet` as a detached
process, so the agent never waits on Jira. `SessionStart` starts one too, at most every 10
minutes. A file lock (`sync.lock`) makes overlapping runs exit immediately.

**Mapping.**

| friction | Jira |
|---|---|
| item | issue in the configured project with the `friction` label |
| detail, workaround, fix note | description paragraphs under `Workaround` and `Fix` headings |
| hit count, time wasted, regressions | a footer line below a rule in the description |
| occurrence | a comment: a readable line, the note, then a `[friction:hit uid=... ts=... ...]` token |
| resolved / wontfix | a transition to a done-category status; wontfix also adds the `friction-wontfix` label |

**Pull, then push.**

1. **Pull.** Search for issues with the label updated since the last pull (plus 10 minutes of
   slack; everything on the first run). New issues become local items. For known ones, title,
   detail, workaround and fix note are taken from Jira unless the local item has unpushed content
   changes, and status likewise. Every comment with a hit token whose `uid` is not already local
   becomes an occurrence with `source = 'remote'`.
2. **Push.** For each item that has no issue yet, has dirty flags, or has unpushed occurrences:
   create the issue if needed, post one comment per unpushed occurrence, update the description
   and wontfix label, and move the issue to a done or not-done status as needed. A merged item
   gets a "Merged into ..." comment and is closed.

**Conflicts.** Local changes set bits in `items.dirty` (1 = stats, 2 = content, 4 = status). A
pull never overwrites a field group whose bit is set, so **your local edit wins** for fields you
changed. After a push, only the bits that were sent are cleared, in case the item changed while
the request was in flight.

**Failures.** If Jira is unreachable the run stops and everything stays flagged for the next one.
The last error is stored in `meta.sync_error` and shown by `friction sync --check`.

**The token** is read from, in order: the environment variable named by `token_env` (default
`FRICTION_JIRA_TOKEN`), the output of `token_cmd`, then `token_file`. It must be a personal API
token (`ATATT...`); scoped tokens also need `cloud_id`, which routes calls through
`api.atlassian.com`.

## 9. The database

See the [schema diagram](diagrams/schema-erd.html).

| Table | Holds | Kept |
|---|---|---|
| `events` | one row per tool call: start time, tool, summary, ok, duration, error, `source` (`hook` or `transcript`) | 30 days |
| `sessions` | one row per agent session, plus `mark_event` (logged up to) and `nudge_event` (asked about up to) | 30 days after last seen |
| `items` | the friction items | forever |
| `occurrences` | one row per hit: item, session, agent, project, `wasted_s`, `timing`, event window, note, `source` (`live`, `backfill`, `remote`) | forever |
| `items_fts` | FTS5 index over item text, rebuilt per item on every change | follows `items` |
| `backfill_sessions` | which transcripts were processed, at what mtime, with what result | forever |
| `meta` | key/value state: `last_pull`, `last_sync`, `sync_error`, `sync_kick` | forever |

Occurrences outlive the events they were measured from, so `friction show <id> --events` can only
show the tool calls behind an occurrence for 30 days.

Several processes write at once (hooks from parallel sessions, the CLI, the backfill, sync). WAL
mode and a 5 second busy timeout handle that. Schema changes after the first release are listed
in `MIGRATIONS` in `friction.py` and applied in order, tracked by `PRAGMA user_version`.

## 10. Install, update, uninstall

`install.sh` clones (or `git pull --ff-only`s) the repo into `~/.local/share/friction/app` and runs
`installer.py`. Run from a checkout, it installs that checkout instead. The installer:

1. checks for Python 3.8+ and SQLite with FTS5
2. links `friction` into `~/.local/bin` and `SKILL.md` into each skills directory
3. merges hook entries into `~/.claude/settings.json` and `~/.codex/hooks.json` for agents that are
   installed, saving a `.bak` copy first and recognising its own entries by `friction_hook.py`
4. writes and starts the services: the web view (restarted on failure) and the 03:30 backfill
   timer, with a `PATH` that can find the `claude` CLI

It is safe to re-run, and re-running updates. `--dry-run` shows the changes; `--uninstall`
removes links, hooks and services but leaves the database.

## Troubleshooting

| Symptom | Check |
|---|---|
| Sessions do not mention friction at start | the hook entries in `~/.claude/settings.json` or `~/.codex/hooks.json`; start a new session after installing |
| `friction log` says "No recorded events" and the time is untimed | `friction timeline`: if empty, the hook is not recording this agent; pass `--wasted` for now |
| The wrong session gets the occurrence | pass `--session <id>`, or set `FRICTION_SESSION` |
| Backfill never files anything | `friction backfill --since 7d --dry-run`; Linux logs: `journalctl --user -u friction-backfill`; check `claude` is on the service `PATH` |
| Web view down | `systemctl --user status friction-web` (Linux) or `~/Library/Logs/friction-web.log` (macOS) |
| Jira not updating | `friction sync --check` shows whether it is on, the connection, what is pending and the last error |

## Tests

```sh
python3 -m unittest test_friction test_tracker
```

`test_friction.py` covers redaction, search, timing windows (streaks, idle gaps, user waits, `--from`,
estimates), regressions, merge, pruning, the hook events and Claude transcript parsing.
`test_tracker.py` runs sync against an in-memory fake Jira, including two machines sharing one
project, status changes both ways, offline changes and the off-by-default switch. CI runs both on Linux and macOS with
Python 3.11 and the latest 3.x, dry-runs the installer, and on macOS does a real install, checks
the LaunchAgents and the web view, then uninstalls.
