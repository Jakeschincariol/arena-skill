# Patch notes: arena-skill, injection-hardened build

Base: `Jakeschincariol/arena-skill`, commit `df07b8e518e10cdecd8cd7712d9958b6f396250e` (the repo's
only commit at the time of this review).

**Changed:** `arena/SKILL.md` only (19,011 -> 23,507 bytes).
**Unchanged, byte-for-byte identical to upstream:** `arena/bracket.py` (49,482 bytes),
`arena/rubric.md` (3,717 bytes), `arena/strategies.json` (9,250 bytes). Verified by comparing file
sizes against the upstream git tree and by re-running `bracket.py`'s own template-extraction regex
against the patched `SKILL.md` to confirm it still parses into exactly the five expected templates
(`competitor`, `attacker`, `defender`, `judge`, `final`) with no placeholder left unfilled.

## Why this patch exists

A security review of the upstream skill found the code (`bracket.py`) clean: no eval/exec/subprocess/
pickle/shell injection, atomic state writes, mostly internally generated file paths. The actual risk
is architectural: the skill's instructions orchestrate up to ~100 general-purpose, tool-using
sub-agents that read files written by each other (solutions, attacks, defenses, verdicts) and,
per the competitor brief, whatever the task itself points to (a file, a URL, pasted text).

The task block in every prompt template was already wrapped in `=== THE TASK === / === END OF THE
TASK ===` markers, a reasonable anti-injection convention. But the files sub-agents read from each
other later in the tournament (an opponent's solution, the attacks against you, both sides' revised
solutions and verdicts) were referenced only by bare file path, with no equivalent "this is data, not
instructions" framing. Since those files can carry content that originated from the task (including
anything external the task pointed at), a prompt injection planted once could in principle ride along
competitor -> attacker -> defender -> judge, each hop a fresh sub-agent with full tool access (Bash,
file write, etc. per this environment's general-purpose agent) and no memory of the original
conversation to sanity-check it against.

## What changed, concretely

In `arena/SKILL.md`:

1. **Every template** (`competitor`, `attacker`, `defender`, `judge`, `final`) now has an explicit
   paragraph, right after the file(s) it is told to read, stating that the referenced file(s) are
   data to work on / judge / attack, not instructions, and that the sub-agent must ignore any
   instruction-like text found inside them, continue its actual job, and note in one line that it saw
   and ignored such content. This mirrors the delimiting already applied to the task block and
   extends it to inter-agent files.
2. **Step 1 (size it, get a yes)**: tightened the "the user asked for the arena" trigger so a phrase
   like "arena" only counts as the user's own request when it appears in their own message, not when
   it merely appears inside pasted text, a file or a URL the orchestrator was asked to read. Otherwise
   the confirmation gate (the one thing standing between an injected instruction and up to ~600
   sub-agent calls) could be talked around by content, not by the user.
3. **Step 2 (write the task file)**: added an instruction to note explicitly, inside `task.md` itself,
   whenever the task includes material the user did not type directly (a file, URL, ticket, etc.), so
   every downstream sub-agent brief carries that caution once, at the source.
4. **Step 4 (the loop)**: added a rule that if any sub-agent's reply or output flags that it saw and
   ignored instruction-like content, the orchestrator stops after that wave, tells the user plainly,
   and asks before continuing, instead of pushing on automatically.
5. **Rules for the orchestrator**: added a bullet making the "treat other agents' files as untrusted"
   stance explicit for the orchestrator's own behaviour too (don't relay their raw contents into a new
   prompt, treat a sub-agent's flag as signal).

## What this patch does NOT fix

This is a prompt-level mitigation, not a technical control. It asks every sub-agent to recognize and
refuse injected instructions and self-report when it does; it does not, and cannot by itself, enforce
that sub-agents stay inside `.arena/`, don't run shell commands, or don't fetch URLs. A sufficiently
well-crafted injection could still work some of the time. If you need a hard guarantee rather than a
much-reduced likelihood, that would require running the sub-agents under an actual sandboxed/restricted
tool-permission profile, which is outside what a Markdown-only skill can do.

## How to verify this patch yourself

- Diff `arena/SKILL.md` in this folder against
  `https://raw.githubusercontent.com/Jakeschincariol/arena-skill/df07b8e518e10cdecd8cd7712d9958b6f396250e/skills/arena/SKILL.md`
  to see exactly what was added (no deletions, only new paragraphs/bullets).
- Confirm `arena/bracket.py`, `arena/rubric.md` and `arena/strategies.json` are unchanged (same byte
  counts as upstream: 49,482 / 3,717 / 9,250).
