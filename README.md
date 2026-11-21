# claude-sessions

[![CI](https://github.com/lucaslobasso/claude-sessions/actions/workflows/ci.yml/badge.svg)](https://github.com/lucaslobasso/claude-sessions/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20macOS%20%7C%20Linux-lightgrey.svg)](#requirements)
[![Claude Code plugins](https://img.shields.io/badge/Claude%20Code-plugins-d97757.svg)](#install)

<p align="center">
  <img src="docs/status.png" alt="Screenshot: /limit-revive status showing the mode, threshold, delay, the 5-hour window at 55% and nothing scheduled" width="760">
</p>

Three Claude Code plugins that keep long sessions going without you watching them:

| Plugin | Version | What it does |
| --- | --- | --- |
| [**limit-revive**](#limit-revive) | 2.0.0 | Resumes unfinished work after the **5-hour** limit resets, and after the **weekly** limit when its reset is less than 12 hours away. With the weekly reset further off, it stops at the threshold instead. |
| [**context-handoff**](#context-handoff) | 1.0.0 | When the context window passes a threshold, Claude writes a handoff file and a **fresh session** picks it up on the same model in auto mode. |
| [**subagentify**](#subagentify) | 1.0.0 | When Claude plans, the work is split into **subagents**, each on **Opus or Sonnet** as the task requires. |

Formerly `claude-limit-revive`. Old links redirect here.

> Unofficial community project, not affiliated with Anthropic.

**[Requirements](#requirements) · [Install](#install) · [limit-revive](#limit-revive) · [context-handoff](#context-handoff) · [subagentify](#subagentify) · [Update](#update) · [Troubleshooting](#troubleshooting) · [Uninstall](#uninstall) · [Install the whole set](#install-the-whole-set)**

## Requirements

- Claude Code **2.1.3xx or newer** (`claude --version`), which has the function-hook plugin API (early access; it may
  change between releases).
- limit-revive: a Claude subscription with usage windows (Pro or Max). With API billing there are none and it stays
  idle.
- Windows, macOS or Linux. The Windows Task Scheduler fallback and the automatic handoff launch use Windows Terminal
  (`wt.exe`) when it is installed. On macOS and Linux, context-handoff prints the command to start the new session
  instead.

## Install

In a Claude Code session, pick any or all:

```
/plugin install limit-revive --marketplace lucaslobasso/claude-sessions
/plugin install context-handoff --marketplace lucaslobasso/claude-sessions
/plugin install subagentify --marketplace lucaslobasso/claude-sessions
```

Answer `y` to add the marketplace and pick the **user** scope (every project). Or from a shell:

```sh
claude plugin marketplace add lucaslobasso/claude-sessions
claude plugin install limit-revive@claude-sessions --scope user
claude plugin install context-handoff@claude-sessions --scope user
claude plugin install subagentify@claude-sessions --scope user
```

They work from the next session on. Settings live in `/config` under each plugin's name, or at install time with
`--config key=value`.

**Coming from claude-limit-revive?** Swap the old install for the new one:

```sh
claude plugin uninstall limit-revive@claude-limit-revive
claude plugin marketplace remove claude-limit-revive
claude plugin marketplace add lucaslobasso/claude-sessions
claude plugin install limit-revive@claude-sessions --scope user
```

## limit-revive

When a usage limit is close **and there is unfinished work**, limit-revive schedules a resume for the reset time plus a
few minutes. At that time it sends "continue where you left off" into the same session. Work that already finished is
never "continued".

- **5-hour window:** always revived after the reset.
- **Weekly window:** revived the same way when the reset is within `weeklyReviveWithinHours` (12). With the reset
  further away, the session **stops at the threshold** according to `weeklyMode`, and nothing is scheduled.
- The 5-hour revive checks the weekly window first, so a session is never resumed into an exhausted week.
- Claude can call it itself through the `limit_revive` tool (`status`, `arm`, `cancel`). When a threshold is crossed,
  Claude gets one note asking it to wrap up at a point it can resume from.

| Setting | Default | Values | What it does |
| --- | --- | --- | --- |
| `thresholdPercent` | `90` | 0 to 100 | 5-hour usage percent at which the plugin gets involved |
| `delayMinutes` | `5` | minutes | How long after a reset to resume |
| `mode` | `resume-only` | `resume-only` / `finish-turn` / `stop-now` | How a revive is set up (below) |
| `resumePrompt` | `The usage limit has reset. Continue exactly where you left off.` | text | Sent to the session when it resumes |
| `windowsFallback` | `true` | true / false | Windows only: also schedule a task that reopens the session if its window was closed |
| `weeklyEnabled` | `true` | true / false | Watch the weekly window too |
| `weeklyThresholdPercent` | `95` | 0 to 100 | Weekly usage percent at which the plugin gets involved |
| `weeklyReviveWithinHours` | `12` | hours | A weekly reset closer than this is revived; a later one is a stop |
| `weeklyMode` | `finish-turn` | `finish-turn` / `stop-now` / `notify-only` | What a weekly stop does |

**Modes:**

- `resume-only` (default): never interrupts. Only a turn that died at the limit is resumed.
- `finish-turn`: lets the running turn finish, holds prompts you type over the threshold, and sends them after the
  reset.
- `stop-now`: stops the running turn at the threshold to save what is left of the window, and resumes it after the
  reset.

**Weekly modes** (reset more than `weeklyReviveWithinHours` away, nothing scheduled):

- `finish-turn` (default): lets the running turn finish and turns away new prompts until the reset (or until you run
  `/limit-revive cancel`).
- `stop-now`: stops the running turn at the threshold.
- `notify-only`: a notice for you and for Claude, nothing else.

| Command | What it does |
| --- | --- |
| `/limit-revive status` | Settings, the 5-hour and weekly readings, and what is scheduled |
| `/limit-revive cancel` | Drops the scheduled resume (and any held prompt). The plugin stays out of the way until the window resets |
| `/limit-revive now` | Resumes right away instead of waiting |

While a resume is scheduled, the status bar shows `⏸ revive at 14:05 (turn stopped at the limit)`. During a weekly stop
it shows `⏹ weekly limit, resets Mon 09:00`.

<details>
<summary>How it works, and the Windows fallback</summary>

1. Every usage reading (`session.measure`) updates the 5-hour and weekly percent and reset time.
2. When a turn ends at the limit, or a threshold is passed in `finish-turn` / `stop-now` mode, the plugin arms a timer
   for **reset + `delayMinutes`** and shows it in the status bar.
3. A turn that finishes normally, or a prompt you type, cancels the timer: there is nothing left to continue.
4. When the timer fires and the window really has reset, it submits `resumePrompt` (or your held prompt). If usage is
   still high with a new reset time, it re-arms instead.

With `windowsFallback` on, Windows also gets a one-shot Task Scheduler job (`ClaudeLimitRevive_<id>`) 2 minutes after
the in-session timer. It opens Windows Terminal with `claude --resume <session> "<prompt>"`, so the session comes back
even if its window was closed. A background session (agent view, `claude --bg`) continues in the background instead,
with `claude --bg --resume`. Claude Code retires an idle background session after about an hour, so background
sessions depend on this fallback.

</details>

<p align="center">
  <img src="docs/status-stop-now.png" alt="/limit-revive status with stop-now mode, 80% threshold and a 2 minute delay" width="560">
</p>

## context-handoff

Long sessions get slower and lose detail once the context fills up and is compacted. context-handoff steps in before
that happens:

1. Context fill passes `thresholdPercent` (75).
2. Depending on `mode`, Claude finishes the turn (`finish-turn`) or stops right away (`stop-now`). Then it writes a
   self-contained handoff file to `~/.claude/handoffs/<date>-<session>.md`: goal, what's done, what's in progress,
   next steps, key files, decisions, open questions, how to verify, repo and branch.
3. The `context_handoff` tool starts a **new session in a new Windows Terminal tab**, on the **same model** with
   `--permission-mode auto`. A background session starts the new one in the background (`claude --bg`).
4. The new session reads the handoff file, **deletes it**, and continues.

You can also ask Claude to "hand off to a new session" at any time.

| Setting | Default | Values | What it does |
| --- | --- | --- | --- |
| `thresholdPercent` | `75` | 0 to 100 | Context fill that triggers the handoff. Keep it below auto-compact |
| `mode` | `finish-turn` | `finish-turn` / `stop-now` / `notify-only` | When to hand off; `notify-only` just tells you and Claude |
| `handoffDir` | `~/.claude/handoffs` | folder | Where handoff files are written |
| `permissionMode` | `auto` | any `--permission-mode` value | Permission mode of the new session |
| `sameModel` | `true` | true / false | Start the new session on the current model |
| `autoLaunch` | `true` | true / false | Launch the new session, or only show the command to run |

| Command | What it does |
| --- | --- |
| `/context-handoff status` | Threshold, current context fill, and handoff state |
| `/context-handoff now` | Hand off right away |
| `/context-handoff cancel` | Stop this handoff. It won't trigger again until context drops below the threshold |

## subagentify

Whenever Claude plans (in plan mode or any multi-step plan), subagentify has it split the work into independent units
and give each one to a subagent with an explicit model:

| Model | For |
| --- | --- |
| **opus** | Architecture and design, ambiguous debugging, security, cross-cutting refactors: anything that needs judgment |
| **sonnet** | Well-specified implementation, tests, docs, mechanical edits, search |

Independent units run in parallel. The main thread integrates and verifies. The rule is added to the system prompt, and
the `subagentify` skill has the full version. If a subagent is spawned without opus or sonnet (no model, or haiku), a
quick Haiku classifier reads the task and picks one. Claude's own explicit choice is never overridden, and forks are
left alone.

| Setting | Default | Values | What it does |
| --- | --- | --- | --- |
| `enabled` | `true` | true / false | Turn the rule and the spawn hook on or off |
| `classifyFallback` | `true` | true / false | Pick opus/sonnet with a classifier when a spawn names neither |
| `defaultModel` | `sonnet` | `opus` / `sonnet` | Used when the classifier is off or can't decide |

## Update

```sh
claude plugin marketplace update claude-sessions
claude plugin update limit-revive@claude-sessions
claude plugin update context-handoff@claude-sessions
claude plugin update subagentify@claude-sessions
```

See [CHANGELOG.md](CHANGELOG.md) for what changed.

## Troubleshooting

- **`/limit-revive` or `/context-handoff` is an unknown command.** Start a new session after installing, and check that
  `claude plugin list` shows the plugin enabled.
- **"5-hour window: no reading yet".** Claude Code reports usage after the first reply of a session. Send any prompt.
- **It didn't resume.** It only resumes unfinished work: a turn that finished normally cancels the timer by design.
  `/limit-revive status` shows what is scheduled.
- **"Windows task not scheduled".** PowerShell or Task Scheduler refused the job (some managed PCs block it), or
  `claude` is not on your PATH. The in-session timer still works while the window stays open. Set `windowsFallback`
  to false to stop the notice.
- **A session opened by itself.** That is the Windows fallback reviving a session whose window was closed, or a
  context handoff. Turn off `windowsFallback` or `autoLaunch` if you don't want it.
- **The handoff session isn't in auto mode.** If your Claude Code doesn't allow `--permission-mode auto`, the launch is
  retried without it and the tool says so.

## Uninstall

```sh
claude plugin uninstall limit-revive@claude-sessions
claude plugin uninstall context-handoff@claude-sessions
claude plugin uninstall subagentify@claude-sessions
claude plugin marketplace remove claude-sessions
```

A Windows task that is already scheduled expires by itself; to remove it now:
`Get-ScheduledTask ClaudeLimitRevive_* | Unregister-ScheduledTask -Confirm:$false`. Handoff files left in
`~/.claude/handoffs` can be deleted.

<!-- bundle:start (copied from claude-launcher/docs/bundle-section.md; edit it there and copy to all six READMEs) -->
## Install the whole set

This is one of six small tools that work well together. Each installs on its own, or get them all with one command:

| Tool | What it does | Runs on |
| --- | --- | --- |
| [Claude Launcher](https://github.com/lucaslobasso/claude-launcher) | One click from the taskbar or Start menu straight into Claude Code, with a Claude icon and a loading spinner | Windows, macOS |
| [Claude Taskbar](https://github.com/lucaslobasso/claude-taskbar) | Your Claude sessions and usage limits as dots and bars on the Windows taskbar | Windows |
| [boxed-messages](https://github.com/lucaslobasso/claude-boxed-messages) | Colored boxes around your prompts and Claude's replies | Any |
| [claude-statusline](https://github.com/lucaslobasso/claude-statusline) | A status line with model, context, branch, and usage only when it gets high | Any |
| [claude-sessions](https://github.com/lucaslobasso/claude-sessions) | limit-revive resumes unfinished work after the 5-hour or weekly limit resets, context-handoff continues in a fresh session before the context fills, subagentify splits planned work across Opus and Sonnet subagents | Any |
| [language-spinner](https://github.com/lucaslobasso/claude-language-spinner) | The "thinking" spinner shows words in a language you are learning | Any |

**Windows** (PowerShell 7):

```powershell
irm https://raw.githubusercontent.com/lucaslobasso/claude-launcher/main/bundle.ps1 | iex
```

To pick, pass options: `-Only statusline,limit-revive`, `-Skip launcher`, `-List` (show what is installed) or `-Uninstall`:

```powershell
& ([scriptblock]::Create((irm https://raw.githubusercontent.com/lucaslobasso/claude-launcher/main/bundle.ps1))) -Skip taskbar
```

**macOS and Linux:** the plugins work everywhere. On macOS the launcher installs with its own one-liner:

```sh
curl -fsSL https://raw.githubusercontent.com/lucaslobasso/claude-launcher/main/bootstrap.sh | bash
```

The plugins, in a shell:

```sh
for p in boxed-messages statusline language-spinner; do
  claude plugin marketplace add "lucaslobasso/claude-$p"
  claude plugin install "$p@claude-$p" --scope user
done
claude plugin marketplace add lucaslobasso/claude-sessions
for p in limit-revive context-handoff subagentify; do
  claude plugin install "$p@claude-sessions" --scope user
done
```
<!-- bundle:end -->

## Contributing

```sh
git clone https://github.com/lucaslobasso/claude-sessions
cd claude-sessions
claude --plugin-dir plugins/limit-revive         # try one plugin's changes in a session
claude plugin test plugins/limit-revive          # its tests (hooks/*.test.ts)
claude plugin validate plugins/limit-revive      # its manifest and hooks
claude plugin validate .claude-plugin/marketplace.json
```

- Loading a plugin once writes its type declarations to `plugins/<name>/.claude-plugin/types/` (git-ignored), which its
  `tsconfig.json` uses for editor type checking.
- `python docs/screenshots.py` retakes the screenshots offscreen (needs `pip install pyte pillow pywinpty`; uses your
  login in a throwaway config folder).

## License

[MIT](LICENSE) © Lucas Lobasso
