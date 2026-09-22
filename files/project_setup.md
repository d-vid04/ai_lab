# project_setup.md

Everything needed to set this project up from scratch on a new machine and run it.
The agent keeps this file current whenever setup steps change.

## 1. What you need

| Tool | Why | Check it's installed |
|---|---|---|
| A modern browser (Chrome, Edge, Firefox, Safari) | Runs the game | — |
| Git | Version control and safety commits | `git --version` |
| Claude Code | The coding agent (used instead of Codex for this lab) | `claude --version` |
| VS Code (optional) | Editor, terminals, Claude Code extension | `code --version` |
| A Claude account with Claude Code access (Pro, Max, Team, or Enterprise) or an API key | To log in to Claude Code | — |

No Node, Python, or build tools are required to **run** the game. It is one HTML file.

## 2. Install Claude Code

Use the official installer for your OS (current instructions:
https://docs.claude.com/en/docs/claude-code/overview).

- **macOS / Linux / WSL**
  ```bash
  curl -fsSL https://claude.ai/install.sh | bash
  ```
- **Windows (PowerShell)**
  ```powershell
  irm https://claude.ai/install.ps1 | iex
  ```

Then open a new terminal and run `claude`. The first launch asks you to log in in the browser.

**VS Code (optional):** Extensions tab → search "Claude Code" (publisher: Anthropic) →
Install. Open the Claude Code panel from the Spark icon in the editor toolbar or the
Command Palette ("Claude Code: Open"). Keep a normal terminal open too, for git.

## 3. Create the project folder

Put the project under **Documents**, not Desktop. On Windows, agents have hit permission
problems in Desktop (OneDrive redirection).

```bash
# Windows (PowerShell)
mkdir $HOME\Documents\AI_Project
cd $HOME\Documents\AI_Project

# macOS / Linux
mkdir -p ~/Documents/AI_Project && cd ~/Documents/AI_Project
```

If you cloned an existing repo, `cd` into it instead and skip to step 5.

## 4. Initialize git

```bash
git init
git config user.name "Your Name"          # if not set globally
git config user.email "uID@umail.utah.edu"
```

Create a `.gitignore`:

```
.DS_Store
Thumbs.db
.vscode/
.claude/settings.local.json
```

## 5. Add the instruction files

The repo root should contain:

```
AI_Project/
├── AGENTS.md          # agent rules (shared by any agent)
├── CLAUDE.md          # makes Claude Code load AGENTS.md
├── instructions.md    # the game spec and milestones
├── project_setup.md   # this file
├── CHANGELOG.md       # created by the agent at v0.1.0
└── cannon.html        # created by the agent, the deliverable
```

**Important:** Claude Code reads `CLAUDE.md`, not `AGENTS.md`. Create `CLAUDE.md` with this
content so Claude Code pulls in the shared rules:

```markdown
@AGENTS.md

## Claude Code notes
- Before each milestone, use plan mode and show me the plan before editing.
- The spec is in @instructions.md. Re-read the relevant section before each milestone.
```

The `@AGENTS.md` line is an import. On first launch Claude Code may ask you to approve the
import; approve it. **Do not run `/init`** in this repo: it would generate its own `CLAUDE.md`
over yours.

Commit the starting state:

```bash
git add .
git commit -m "Project setup: agent instructions and spec"
```

## 6. Start a Claude Code session

From the project folder:

```bash
claude
```

Useful controls (run `/help` for the current full list):

| Action | How |
|---|---|
| Check which model is active / switch | `/model` |
| Confirm CLAUDE.md + AGENTS.md loaded | `/memory` (or ask "Which instruction files do you have loaded?") |
| Plan mode (agent plans, doesn't edit) | `Shift+Tab` to cycle modes |
| Interrupt the agent | `Esc` |
| Add guidance while it works | Just type a message; it's picked up at the next step |
| Start fresh context | `/clear` |
| Check usage / limits | `/usage` or `/status` |

**Permissions:** Claude Code asks before editing files or running commands. If you don't
notice a prompt, it just waits. Leave the default ask-first mode on for this lab; don't enable
auto-accept or skip-permissions until you've used it for a while.

**Parallel sessions:** you can run more than one `claude` session, but never let two of them
edit `cannon.html` at the same time.

## 7. First prompt to give the agent

```
Read AGENTS.md and instructions.md. Summarize the rules you'll follow and the milestone plan
in under 15 lines, list any questions about the spec, then stop. Don't write code yet.
```

After that, for each milestone:

```
Start milestone M<n> from instructions.md. Plan first, then implement, then stop and report
per AGENTS.md.
```

## 8. Running the game

- Double-click `cannon.html`, or drag it into a browser window.
- Self-tests: open the file, add `#test` to the end of the URL, and press Enter
  (e.g. `file:///C:/Users/you/Documents/AI_Project/cannon.html#test`), then reload.
  Results appear in an on-page panel and in the browser console (F12 → Console).
- Optional local server (only if a browser blocks something on `file://`):
  `python -m http.server 8000` then open http://localhost:8000/cannon.html

## 9. End-of-session routine

1. Ask the agent: *"Give me the commit-log bullets for this session."*
2. Play the game once end to end and run `#test`.
3. `git add . && git commit -m "<first bullet>" -m "<remaining bullets>"`

## 10. Submission checklist

- [ ] `cannon.html` opens and plays with no console errors
- [ ] Header comment filled in: name, uID, date, class, agent + model, other AI used
- [ ] `#test` shows all self-tests passing
- [ ] Version number in header area/footer matches `CHANGELOG.md`
- [ ] Final commit made
- [ ] Upload **only** `cannon.html` to Canvas
