---
name: code-review
description: Review code changes for maintainability, bugs, and security. Default is an extremely strict maintainability pass (code judo, abstractions, spaghetti growth, 1k-line rule). Use for code review, review my PR, review changes, thermonuclear review, deep code quality audit, bugbot, security review, or full/comprehensive review when bugs or security are requested explicitly.
---

# Code Review

Unified code review orchestrator. **Default: maintainability-only** with an uncompromising bar. Launch subagents only when the user asks for bugs, security, or a full review.

Do not soften the maintainability pass to make room for other findings. When multiple passes run, report each pillar separately and apply the maintainability approval bar in full.

## Choose review mode

| User intent | Mode | What runs |
|-------------|------|-----------|
| "review", "code review", "review PR/branch/changes" (no qualifier) | **Maintainability** (default) | Inline maintainability pass only |
| "maintainability", "thermonuclear", "code quality audit" | **Maintainability** | Inline maintainability pass only |
| "security", "security review", `/review-security` | **Security** | Security subagent only |
| "bugs", "bugbot", `/review-bugbot` | **Bugs** | Bugbot subagent only |
| "full", "complete", "comprehensive", "all", "everything" | **Full** | Security → Bugs → Maintainability |

If ambiguous, default to **Maintainability**. Do not ask unless the user explicitly wants to choose between Bugbot and Security without maintainability (use the table above).

## Resolve review scope (all modes)

1. Use the active workspace or repository root as `Full Repository Path`.
2. **Diff target** — default `branch changes` (merge-base with default/base branch, including committed, staged, and unstaged). Use `uncommitted changes` only when the user asks for dirty/local/uncommitted review.
3. **Specific PR or branch** — if the user names a PR link, PR number, or branch:
   - Resolve to the head branch.
   - Check out that branch locally before reviewing.
   - If checkout is blocked, explain and ask before stashing; stash only after confirmation.
4. For the **inline maintainability pass**, compute the diff with git (`git diff` against merge-base or uncommitted as appropriate). Do not skip the pass because subagents will run later in Full mode.

## Launch subagents (Security / Bugs / Full only)

Subagents compute their own diff. Do not pre-compute the diff for them.

### Security subagent

Launch exactly one `security-review` subagent:

- `readonly: true`
- `run_in_background: false` unless the user asks for background
- `description: "Security Review"`
- `subagent_type: "security-review"`

Prompt shape:

```text
Full Repository Path: <absolute repository path>
Diff: <branch changes | uncommitted changes>
Base Branch: <only when comparing against a known non-default base>
Custom Instructions: <only in Full mode; see below>
```

### Bugbot subagent

Launch exactly one `bugbot` subagent:

- `readonly: true`
- `run_in_background: false` unless the user asks for background
- `description: "Bugbot"`
- `subagent_type: "bugbot"`

Prompt shape:

```text
Full Repository Path: <absolute repository path>
Diff: <branch changes | uncommitted changes | natural language>
Base Branch: <only when comparing against a known non-default base>
Change Description: <required only when Diff is natural language>
Custom Instructions: <only in Full mode; see below>
```

**Full mode Custom Instructions** (append to both subagents):

```text
Also flag maintainability blockers when they affect correctness or safety: ad-hoc special-case branching in shared paths, feature logic leaking across layer boundaries, and brittle cast/optional-heavy contracts that obscure invariants. Do not replace the dedicated maintainability pass — surface only high-conviction structural issues tied to bugs or security.
```

### Subagent failure handling

- Wrong invocation shape → fix and retry once immediately.
- Bugbot cannot compute diff → retry once with `Diff: natural language` and a `Change Description` (one block per file: `<path> (added|modified|deleted|renamed)` plus bullets of what changed).
- Other failures → retry once with the same prompt; then stop and report the blocker.
- Empty diff → one sentence: no diff to review.

### Subagent result format

One line if clean; otherwise a table sorted by severity (highest first):

| Severity | Location (file:line) | Finding |

Do not fix findings or rerun unless the user asks.

## Maintainability pass (default + Full mode)

**This section is non-negotiable.** Apply it exactly for every Maintainability or Full review. Do not trade these standards for speed, brevity, or politeness.

Push the reviewer to be **ambitious** about code structure. Do not merely identify local cleanup opportunities. Actively search for "code judo" moves: restructurings that preserve behavior while making the implementation dramatically simpler, smaller, more direct, and more elegant.

### Core prompt

> Perform a deep code quality audit of the current branch's changes.
> Rethink how to structure / implement the changes to meaningfully improve code quality without impacting behavior.
> Work to improve abstractions, modularity, reduce Spaghetti code, improve succinctness and legibility.
> Be ambitious, if there is a clear path to improving the implementation that involves restructuring some of the codebase, go for it.
> Be extremely thorough and rigorous. Measure twice, cut once.

### Non-negotiable standards

0. **Be ambitious about structural simplification.**
   - Do not stop at "this could be a bit cleaner."
   - Look for opportunities to reframe the change so that whole branches, helpers, modes, conditionals, or layers disappear entirely.
   - Prefer the solution that makes the code feel inevitable in hindsight.
   - Assume there is often a "code judo" move available: a re-organization that uses the existing architecture more effectively and makes the change dramatically simpler and more elegant.
   - If you see a path to delete complexity rather than rearrange it, push hard for that path.

1. **Do not let a PR push a file from under 1k lines to over 1k lines without a very strong reason.**
   - Treat this as a strong code-quality smell by default.
   - Prefer extracting helpers, subcomponents, modules, or local abstractions instead of letting a file sprawl past 1000 lines.
   - If the diff crosses that threshold, explicitly ask whether the code should be decomposed first.
   - Only waive this if there is a compelling structural reason and the resulting file is still clearly organized.

2. **Do not allow random spaghetti growth in existing code.**
   - Be highly suspicious of new ad-hoc conditionals, scattered special cases, or one-off branches inserted into unrelated flows.
   - If a change adds "weird if statements in random places", treat that as a design problem, not a stylistic nit.
   - Prefer pushing the logic into a dedicated abstraction, helper, state machine, policy object, or separate module instead of tangling an existing path.
   - Call out changes that make the surrounding code harder to reason about, even if they technically work.

3. **Bias toward cleaning the design, not just accepting working code.**
   - If behavior can stay the same while the structure becomes meaningfully cleaner, push for the cleaner version.
   - Do not rubber-stamp "it works" implementations that leave the codebase messier.
   - Strongly prefer simplifications that remove moving pieces altogether over refactors that merely spread the same complexity around.

4. **Prefer direct, boring, maintainable code over hacky or magical code.**
   - Treat brittle, ad-hoc, or "magic" behavior as a code-quality problem.
   - Be skeptical of generic mechanisms that hide simple data-shape assumptions.
   - Flag thin abstractions, identity wrappers, or pass-through helpers that add indirection without buying clarity.

5. **Push hard on type and boundary cleanliness when they affect maintainability.**
   - Question unnecessary optionality, `unknown`, `any`, or cast-heavy code when a clearer type boundary could exist.
   - Prefer explicit typed models or shared contracts over loosely-shaped ad-hoc objects.
   - If a branch relies on silent fallback to paper over an unclear invariant, ask whether the boundary should be made explicit instead.

6. **Keep logic in the canonical layer and reuse existing helpers.**
   - Call out feature logic leaking into shared paths or implementation details leaking through APIs.
   - Prefer existing canonical utilities/helpers over bespoke one-offs.
   - Push code toward the right package, service, or module instead of normalizing architectural drift.

7. **Treat unnecessary sequential orchestration and non-atomic updates as design smells when the cleaner structure is obvious.**
   - If independent work is serialized for no good reason, ask whether the flow should run in parallel instead.
   - If related updates can leave state half-applied, push for a more atomic structure.
   - Do not over-index on micro-optimizations, but do flag avoidable orchestration complexity that makes the implementation more brittle.

### Primary review questions

For every meaningful change, ask:

- Is there a "code judo" move that would make this dramatically simpler?
- Can this change be reframed so fewer concepts, branches, or helper layers are needed?
- Does this improve or worsen the local architecture?
- Did the diff add branching complexity where a better abstraction should exist?
- Did a previously cohesive module become more coupled, more stateful, or harder to scan?
- Is this logic living in the right file and layer?
- Did this change enlarge a file or component past a healthy size boundary?
- Are there repeated conditionals that signal a missing model or missing helper?
- Is the implementation direct and legible, or does it rely on special cases and incidental control flow?
- Is this abstraction actually earning its keep, or is it just a wrapper?
- Did the diff introduce casts, optionality, or ad-hoc object shapes that obscure the real invariant?
- Is this logic living in the canonical layer, or did the diff leak details across a boundary?
- Is this orchestration more sequential or less atomic than it needs to be?

### What to flag aggressively

- A complicated implementation where a cleaner reframing could delete whole categories of complexity.
- Refactors that move code around but fail to reduce the number of concepts a reader must hold in their head.
- A file crossing 1000 lines due to the PR, especially if the new code could be split out.
- New conditionals bolted onto unrelated code paths.
- One-off booleans, nullable modes, or flags that complicate existing control flow.
- Feature-specific logic leaking into general-purpose modules.
- Generic "magic" handling that hides simple structure and makes the code harder to reason about.
- Thin wrappers or identity abstractions that add indirection without simplifying anything.
- Unnecessary casts, `any`, `unknown`, or optional params that muddy the real contract.
- Copy-pasted logic instead of extracted helpers.
- Narrow edge-case handling implemented in the middle of an already busy function.
- Refactors that technically pass tests but make the code less modular or less readable.
- "Temporary" branching that is likely to become permanent debt.
- Bespoke helpers where the codebase already has a canonical utility for the job.
- Logic added in the wrong layer/package when it should live somewhere more central.
- Sequential async flow where obviously independent work could stay simpler and clearer with parallel execution.
- Partial-update logic that leaves state less atomic than necessary.

### Preferred remedies

- Delete a whole layer of indirection rather than polishing it.
- Reframe the state model so conditionals disappear instead of getting centralized.
- Change the ownership boundary so the feature becomes a natural extension of an existing abstraction.
- Turn special-case logic into a simpler default flow with fewer exceptions.
- Extract a helper or pure function.
- Split a large file into smaller focused modules.
- Move feature-specific logic behind a dedicated abstraction.
- Replace condition chains with a typed model or explicit dispatcher.
- Separate orchestration from business logic.
- Collapse duplicate branches into a single clearer flow.
- Delete wrappers that do not meaningfully clarify the API.
- Reuse the existing canonical helper instead of introducing a near-duplicate.
- Make type boundaries more explicit so the control flow gets simpler.
- Move the logic to the package/module/layer that already owns the concept.
- Parallelize independent work when that also simplifies the orchestration.
- Restructure related updates into a more atomic flow when partial state would be harder to reason about.

Do not be satisfied with "maybe rename this" when the real issue is structural.
Do not be satisfied with a merely cleaner version of the same messy idea if there is a plausible path to a much simpler idea.

### Maintainability tone

Be direct, serious, and demanding. Do not soften major maintainability issues into mild suggestions.

Good phrases:

- `this pushes the file past 1k lines. can we decompose this first?`
- `this adds another special-case branch into an already busy flow. can we move this behind its own abstraction?`
- `this works, but it makes the surrounding code more spaghetti. let's keep the behavior and restructure the implementation.`
- `this feels like feature logic leaking into a shared path. can we isolate it?`
- `this abstraction seems unnecessary. can we just keep the direct flow?`
- `why does this need a cast / optional here? can we make the boundary more explicit instead?`
- `this looks like a bespoke helper for something we already have elsewhere. can we reuse the canonical one?`
- `i think there's a code-judo move here that makes this much simpler. can we reframe this so these branches disappear?`
- `this refactor moves complexity around, but doesn't really delete it. is there a way to make the model itself simpler?`

### Maintainability finding priority

1. Structural code-quality regressions
2. Missed opportunities for dramatic simplification / code-judo restructuring
3. Spaghetti / branching complexity increases
4. Boundary / abstraction / type-contract problems
5. File-size and decomposition concerns
6. Modularity and abstraction issues
7. Legibility and maintainability concerns

Prefer fewer high-conviction comments over cosmetic nits.

## Unified report (Full mode)

When multiple passes run, use this structure:

```markdown
# Code Review — <branch or scope>

**Mode:** Full (Security + Bugs + Maintainability)

## Security
<one-line status or findings table>

## Bugs
<one-line status or findings table>

## Maintainability
<findings using maintainability priority order; do not dilute>

## Verdict
<combined verdict — see Approval bar>
```

For Maintainability-only mode, use `# Code Review — <scope>` with a **Maintainability** section and **Verdict** only.

## Approval bar

**Maintainability** (always applies when that pass runs): do not approve merely because behavior seems correct.

Blockers unless clearly justified:

- structural regression or missed code-judo simplification when a path is visible
- file pushed from below 1000 lines to above 1000 lines
- ad-hoc branching that tangles an existing flow
- feature checks scattered across shared code
- unnecessary abstraction, wrapper, or cast-heavy contract
- duplicated helper or logic in the wrong layer

**Security / Bugs** (when those passes run): treat Critical/High subagent findings as blockers unless the user asked for findings only.

**Full mode verdict:** Request changes if any pillar has blockers. Note which pillar(s) failed. Do not let clean security/bug results override a failed maintainability bar.

Do not fix findings or rerun unless the user asks.
