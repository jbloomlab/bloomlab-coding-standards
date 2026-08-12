# bloomlab-coding-standards

Shared coding standards for the lab; includes documents for both humans and AI coding assistants (Claude Code).

## What's in here

| File | Audience | Purpose |
|---|---|---|
| [`BEST_PRACTICES.md`](BEST_PRACTICES.md) | Humans | Human-centered description of lab coding standards and rationale. Read this carefully. |
| [`CLAUDE.md`](CLAUDE.md) | Claude Code | A terse rule sheet loaded into every Claude Code session via import. Kept short on purpose. |
| [`commands/review.md`](commands/review.md) | Claude Code | Structural code review you run locally on your code, which you can also attach to a pull request. |
| [`templates/project-CLAUDE.md`](templates/project-CLAUDE.md) | New projects | Template project-level `CLAUDE.md` that imports the lab standards. |

## How to interact with Claude Code

**Choose the model to fit the task.** Claude Code offers several models that differ in capability and usage cost; switch between them with `/model`. Design or debugging problems are worth the most capable models; routine edits may not be. If you are not sure how the models differ or which to pick, ask Jesse or someone who does.

**For anything complex, ask for a plan.** Claude Code has different modes that you can switch between, including a plan mode. For anything complex, first explain to Claude Code the goal and your ideas for how it might be accomplished, then ask it to make a plan that you review together. Only then should you ask it to implement the plan. Except for routine edits, this approach is better than just having it make changes without first discussing the plan. The reason is that while Claude Code is extremely unlikely to make technical errors in writing code (much less likely than you!), it can often make conceptual or structural errors if it does not understand the task. _It is your job to clearly plan and define the task._

## Installing Claude Code and `gh`

Our lab has Claude Code licenses through HHMI — talk to Jesse if you do not have one.

**Get `~/.local/bin` onto your `PATH` first.** Both tools below install there, and neither will run until your shell looks there. Check whether it already does:

```bash
echo "$PATH" | tr ':' '\n' | grep "$HOME/.local/bin"
```

If that prints nothing, add this line to your `~/.bashrc`, then either open a new terminal or run `source ~/.bashrc`:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

**Claude Code.** Follow the [official install instructions](https://code.claude.com/docs/en/setup); on Linux and macOS that is:

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

This installs a self-contained binary that updates itself from then on. Start it by running `claude`, and authenticate in the browser when prompted.

On the Fred Hutch cluster, if Claude Code misbehaves or exits silently on *rhino*, ssh to *maestro* instead — rhino runs Ubuntu 18.04, below Claude Code's documented minimum of Ubuntu 20.04.

**`gh`.** The GitHub CLI is what lets Claude Code post reviews, open pull requests, and read issues. On a Mac, `brew install gh`. On the cluster, install the [latest release](https://github.com/cli/cli/releases) into `~/.local/bin`:

```bash
V=$(curl -fsSLI -o /dev/null -w '%{url_effective}' https://github.com/cli/cli/releases/latest | sed 's#.*/tag/v##')
curl -fsSL "https://github.com/cli/cli/releases/download/v$V/gh_${V}_linux_amd64.tar.gz" | tar xz -C /tmp
mkdir -p ~/.local/bin && cp "/tmp/gh_${V}_linux_amd64/bin/gh" ~/.local/bin/
gh auth login    # GitHub.com → HTTPS → login with a web browser
```

`gh` is a single static binary with no dependencies, but unlike Claude Code it does not update itself, so re-run the above from time to time.

`gh` is optional, and you can always just ask Claude Code to install and authenticate it for you. Without it, `/review` prints the review and you paste it into a pull request comment yourself. **Pasting is always an acceptable fallback — tooling problems are never a reason to be stuck.**

## Adding these standards to a project

From the root of your project repository:

```bash
git submodule add https://github.com/jbloomlab/bloomlab-coding-standards bloomlab-coding-standards
git commit -m "Add lab coding standards as submodule"
```

Then create a project-level `CLAUDE.md` at the repo root (copy [`templates/project-CLAUDE.md`](templates/project-CLAUDE.md) as a starting point). Its first line imports the lab rules:

```markdown
@bloomlab-coding-standards/CLAUDE.md
```

If the project already has a `CLAUDE.md`, just add that import line at the top of it.

Claude Code concatenates memory files, and more specific instructions take precedence, so anything you add below the import can extend or override the lab defaults for that project. Keep `CLAUDE.md` to coding instructions only — what the project is and its scientific context belong in the project's `README.md`.

That's all the setup the review needs: the imported lab rules tell Claude Code where the review instructions live, so in any session you can simply ask — "run the lab review" — and it will follow [`bloomlab-coding-standards/commands/review.md`](commands/review.md).

Optionally, if you want a literal `/review` slash command (with autocomplete), symlink the command file into the project's command directory (the symlink stays in sync when the submodule is updated):

```bash
mkdir -p .claude/commands
ln -s ../../bloomlab-coding-standards/commands/review.md .claude/commands/review.md
```

Optionally, if you want to actually commit this `/review` command into the git history rather than just in your local copy, do:

```bash
git add -f .claude/commands/review.md    # -f: the default .gitignore ignores dotfiles
git commit -m "Add /review slash command"
```

## Updating the standards submodule in your project

Standards are pinned per-project at a specific commit of this repo. To pull the latest standards into a project:

```bash
git submodule update --remote bloomlab-coding-standards
git add bloomlab-coding-standards
git commit -m "Update lab coding standards"
```

When cloning a project that uses the standards:

```bash
git clone --recurse-submodules <project-url>
# or, if already cloned:
git submodule update --init
```

## Changing the standards

The standards are living documents. If a rule is wrong, unclear, or missing, open a pull request against this repository or raise an issue — do not fork project-local variants. A PR that changes a rule must update both [`CLAUDE.md`](CLAUDE.md) (the normative rule set) and [`BEST_PRACTICES.md`](BEST_PRACTICES.md) (the rationale), or state why only one is affected. Discussion happens in the PR; once merged, projects adopt the change the next time they update the submodule.
