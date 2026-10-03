# Code Review Skill

A Cursor Agent skill that runs a strict maintainability review by default, and only launches Security or Bugbot passes when you ask for them.

The default bar is structural: fewer concepts, less spaghetti branching, no quiet drift past the 1k-line file limit, and a bias toward deleting complexity instead of redecorating it.

## Why it exists

"Looks fine" reviews often miss the failures that compound: special-case `if`s bolted onto shared paths, thin wrappers that add indirection without clarity, feature logic leaking into the wrong layer, and refactors that move code without reducing what a reader must hold in their head.

This skill names those failure modes up front and refuses to soften the maintainability pass to make room for other findings. When multiple pillars run, each is reported separately.

## What you get

| Path | Role |
|------|------|
| `SKILL.md` | Full orchestrator: modes, scope rules, subagent prompts, approval bar |

There is no separate tool binary. The skill instructs the Cursor Agent how to review diffs and when to spawn `security-review` or `bugbot` subagents.

## Review modes

| You say | Mode | What runs |
|---------|------|-----------|
| "review", "code review", "review my PR" (no qualifier) | Maintainability | Inline maintainability pass only |
| "maintainability", "thermonuclear", "code quality audit" | Maintainability | Same |
| "security", "security review", `/review-security` | Security | Security subagent only |
| "bugs", "bugbot", `/review-bugbot` | Bugs | Bugbot subagent only |
| "full", "complete", "comprehensive", "everything" | Full | Security → Bugs → Maintainability |

Ambiguous requests default to Maintainability.

Diff scope defaults to branch changes (committed, staged, and unstaged vs merge-base). Use uncommitted-only when you ask for dirty/local review.

## What the maintainability pass blocks

Unless clearly justified, these are approval blockers:

- A clear simpler structure exists and was skipped ("code judo" opportunity)
- A file crosses from under 1,000 lines to over 1,000 without a strong reason
- Ad-hoc branches tangled into an already busy flow
- Feature checks scattered across shared code
- Unnecessary wrappers, casts, or muddy type contracts
- Duplicated helpers or logic in the wrong layer

The pass prefers fewer high-conviction comments over cosmetic nits. It does not auto-fix findings unless you ask.

## Install

1. Copy this folder into your Cursor skills directory, for example:
   - macOS / Linux: `~/.cursor/skills/code-review-skill/`
   - Windows: `%USERPROFILE%\.cursor\skills\code-review-skill\`
2. In Agent chat on a repo with git history, ask for a review in one of the modes above.

## Quick start

Example prompts:

- "Review the current branch changes."
- "Thermonuclear maintainability review of this PR."
- "Full review: security, bugs, and maintainability."
- "Bugbot only on uncommitted changes."

For Full mode, Security and Bugs run first via subagents; Maintainability still runs in full afterward. A clean security/bug result does not override a failed maintainability bar.

## License

MIT. See `LICENSE`.
