# Claude Code orientation

This project lets a Slack Solution Engineer stand up a Slack app that simulates any marketplace AI app (Claude, Asana, Notion AI, Cursor, Glean, etc.) for live demos. The app runs as a long-lived Python process that listens for Slack DMs / @mentions and replies using the SE's **local `claude` CLI** — no API keys in this project.

## Audience posture

Your user is a **Slack Solution Engineer**. Very fluent in Slack (channels, apps, scopes, enterprise grid). **Not a coder.** Never used Python, probably never used Terminal for anything more than copy-pasting one line. **They may also be new to vibe coding (working with Claude Code as a pair-programmer) and to building custom Slack apps end-to-end** — for many SEs, this project will be their first time taking a Slack app from manifest → install → live process. Assume:

- They will not read Python code. Don't tell them to "look at line 42."
- They do not know what a manifest, scope, Socket Mode token, venv, getpass, or subprocess is. **Define every piece of jargon inline the first time it appears.**
- They have used Claude Code before (that's how they're reading this), but may not have used it as a guided builder before — invite them to ask "what does this do?" or "why?" at any step.
- They have their own Slack demo org where they're an admin.

Default to narrating what you're about to do before doing it, pause after each step for confirmation, and when in doubt explain *what* a step does and *why* before running it.

## First-run detection

At the start of every session, **check whether `config/app_config.json` exists**:

- **Missing** → the project has never been configured. Run the **Onboarding Wizard** below.
- **Present** → skip the wizard. Read it to find `display_name`, `model`, and `simulates`, then give a one-line status: *"You're configured as **ClaudeSim**, simulating Claude (model: sonnet). To run it: `./run_app.sh`. Leave that Terminal window open."* Then ask what they want to do (run it, edit persona, reconfigure).

Do not run the wizard if it was already run. To re-run, the SE must explicitly say "start over" or "reconfigure" — then confirm, delete `config/app_config.json`, `manifest.json`, and any persona file under `agent/personas/`, and run the wizard again.

## Onboarding Wizard

The full 10-step wizard (which app to simulate, app name, persona, formatting, taskplan animation, model, demo prompts, vetting, recap) lives in the `onboarding-wizard` skill (`.claude/skills/onboarding-wizard/SKILL.md`) — invoke it whenever first-run detection above says to run the wizard, or when the SE wants to start over / simulate a different app.

---

## Respect `[Claude Code]` vs `[Terminal]` tags in SETUP.md

- **`[Claude Code]` steps** — run the command yourself via the Bash tool and show the result. Safe for non-interactive Python scripts, file edits, JSON reads.
- **`[Terminal] `steps** — **do not run these yourself.** Instead, tell the SE: *"please open a new Terminal window, make sure you're in the project folder, and run this command:"* and give them the exact command. These steps require either (a) hidden input via `getpass` (the token-save scripts), or (b) a long-running process that the SE needs to watch and leave open (`./run_app.sh`). The Bash tool can't drive either of those.

Specifically, **never** try to run these via the Bash tool:
- `tokens/save_bot_token.py`
- `tokens/save_app_token.py`
- `./run_app.sh`

## Venv discipline

Each Bash tool call runs in a fresh shell — `source .venv/bin/activate` doesn't persist. **Always invoke Python as `.venv/bin/python <script>`** (macOS/Linux) or `.venv\Scripts\python.exe <script>` (Windows). Never tell the SE to "activate" the venv when you're about to run something for them.

## Token discipline

**Never ask the SE to paste a bot token or app-level token into chat.** Route them:

- `xoxb-` bot token → `tokens/save_bot_token.py` (getpass; hidden input)
- `xapp-` app-level token → `tokens/save_app_token.py` (getpass; hidden input)

The SE runs these in a Terminal window they open themselves.

`team_id` (starts with `T`) and `audit_channel_id` (starts with `C`) are **not** secrets — fine to capture in chat and write to `tokens.json` for the SE.

## Terminal lifecycle — critical

The simulator is a **long-lived Python process** that runs inside a Terminal window. When the window closes, the process dies, and the bot stops responding in Slack. This is the #1 thing that breaks demos.

**Before every `./run_app.sh`, remind the SE:**

> *"Leave this Terminal window open for the whole demo. Tuck it behind Slack if you want it out of the way, but don't close it. If you close it, the bot stops answering — reopen the window, run `./run_app.sh` again, and it'll come back online."*

The app **does not hot-reload.** Any of these edits require Ctrl+C and re-running `./run_app.sh`:

| Edit | Restart needed? |
|---|---|
| `agent/personas/<slug>.md` (persona edits) | Yes |
| `config/app_config.json` (loading messages, model, blockkit flag) | Yes |
| Python files under `agent/` | Yes |
| `manifest.json` | No — but the Slack app itself needs reinstalling from the new manifest |

When you make any of these edits for the SE, **end your message with an explicit restart reminder**: *"Go to your Terminal window running the app, press Ctrl+C to stop it, then run `./run_app.sh` again. The new changes will load on startup."*

**`manifest.json` edit + Slack client hard-refresh (confirmed live 2026-10-30).** After the SE reinstalls the app from an edited manifest (e.g. a feature block like `agent_view` changed), a UI feature tied to that change may not show up in Slack even though the manifest-level change took effect server-side — the client was showing a cached view. Tell the SE to fully quit Slack (Cmd+Q on Mac, not just closing the window) and relaunch before concluding a manifest-level feature "isn't working."

## What you must NOT do

- **Don't hardcode Anthropic API keys, Bedrock URLs, or bearer tokens.** This project deliberately uses the `claude` CLI so every SE inherits their own auth. If something looks like it needs an API key, stop and check — you're probably on the wrong path.
- **Don't copy `tokens.json`** from `slack-demo-toolkit` or anywhere else. Every SE has their own Slack workspace and their own credentials.
- **Don't edit `agent/demo_agent.py` or `agent/claude_subprocess.py` during onboarding.** If the SE wants different behavior, edit the persona or `config/app_config.json` instead. Only touch the Python if there's a genuine bug.
- **Don't fabricate tool calls or external APIs.** This simulator responds using `claude` + the persona, nothing else. There's no database, no real Asana integration, no live data. The persona tells the bot what to pretend to know.

## Ask Mac or Windows early

Before SETUP.md Step 2 (venv creation), ask which OS the SE is on. The docs are Mac-first; Windows commands are at the bottom of SETUP.md. Swap only the `python3.12`, `source`, and `cp` commands — everything else is identical.

## Audit channel

If the SE set `audit_channel_id` in `tokens.json`, every significant action (startup, scope check, errors, each reply sent) is logged to that channel by `config.audit_log()`. When troubleshooting, **read that channel first** — it has timestamps and specific error strings for every failure, and is much more useful than scrollback from the Terminal window.

If `audit_channel_id` is empty, audit logging silently no-ops. That's fine — the simulator still works.

## If the SE says "I want to simulate a different app"

Safest path: say *"the easiest way is to delete `config/app_config.json` and I'll re-run the wizard from scratch. That keeps the new persona clean. Sound good?"* Wait for confirmation, delete the file, and re-run the wizard.

If they want to keep the same Slack app but just tweak the persona (same `app_name`, same installed Slack app, same tokens, different personality), edit the persona file in place — then remind them to Ctrl+C and restart.
