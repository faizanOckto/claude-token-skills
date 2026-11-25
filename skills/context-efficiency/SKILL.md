---
name: context-efficiency
description: "Apply per-turn tactics to keep token consumption low across coding sessions — constrain bulk tool outputs at the source, avoid re-reading files within a session, prefer Edit over Write for changes, fire independent tool calls in parallel, and don't quote tool results back in prose. Trigger on any multi-step coding work — debugging, refactors, investigations, anything that involves more than a couple of tool calls. These tactics are individually small but compound across a session: a 10% saving per turn becomes a 10% saving on every future turn too. Use alongside the delegate-reading skill when bulk reading is also on the path."
---

# Context efficiency

## The principle

Every token in the working context is reprocessed on every subsequent turn. Tactics that keep the context small and stable compound — saving 10% per turn means saving 10% on every future turn too. Most of these tactics are cheap to apply correctly; the cost of forgetting is hidden because it shows up as a slowly growing per-turn bill, not a single visible spike.

This skill collects the per-turn habits worth keeping. For bulk reading specifically, see the `delegate-reading` skill — same principle, applied to the heaviest case.

## Tactics

### 1. Constrain bulk outputs at the source

Many shell commands return unbounded output by default. A bare `git log`, `find .`, `tree`, broad `grep`, `cat large.log`, `npm ls` can dump thousands of lines straight into context, where they stay for the rest of the session.

Pin every command:

| Default (bad) | Pinned (good) |
|---|---|
| `git log` | `git log --max-count=20 --oneline` |
| `find .` | narrow path + `-maxdepth N` |
| `grep -r pattern` | scope to a dir + `--include='*.kt'`, optionally pipe through `head` |
| `cat large.log` | grep for the relevant line first, then Read with `offset`/`limit` around it (or `tail -100` if the end is what matters) |
| `tree` | `tree -L 2`, or narrow to a subdir |
| `npm ls` | `npm ls --depth=0` |

If the answer is going to be one fact (latest commit message, last error line, top-level package list), the command should return one screen, not the world. A common slip: running an unconstrained command to "just check" something, then dragging the entire output along for the rest of the session.

### 2. Don't re-read the same file in a session

Once a file is in context as a tool result, it stays there. Re-reading the same file later doesn't refresh anything for the model — it appends a second copy, and both copies are then paid for on every later turn.

Before opening Read on a file, scan the conversation for a prior read. If the content is still relevant and the file hasn't been edited since, work from the existing tool result. If it *has* been edited, re-reading is fine — but only if the new content actually matters for the next decision. The Edit tool's success message tells you what changed; trust it for confirmation.

### 3. Edit, not Write, for changes to existing files

With Edit, only the `old_string` / `new_string` pair has to be generated. With Write, the entire new file is generated as output tokens — the most expensive kind — and that tool call then sits in the history as input on every later turn. On a 500-line file with a 3-line change, Edit is roughly two orders of magnitude cheaper.

Reserve Write for genuinely new files or true full-file rewrites. The cost shows up twice: as a slower, pricier response now, and as a much larger conversation history on every turn after.

### 4. Parallelize independent tool calls

When the result of tool A is *not* an input to tool B, fire them in the same message. They execute concurrently. The tool results cost the same, but the conversation has fewer turns — and every turn re-reads the whole context (cached reads are discounted, not free), so three calls in one turn pay for that re-read once instead of three times. It is also faster wall-clock for the user.

Anti-pattern: a chain of single-tool turns when three could have gone out together.

Independence check: if you don't know what A will return, but you know what B needs to do regardless, they're independent. Fire both. Same for reads, greps, and most status commands (`git status` + `git diff` + `git log` is a classic parallel triple).

### 5. Don't quote tool results back in prose

The tool result is already in the context. Restating its content verbatim — "I see that the file contains the following: \[pastes 40 lines\]" — duplicates the content into the assistant message, which gets reprocessed on every future turn.

State conclusions, not the evidence. A one-sentence synthesis ("the auth middleware is at AuthFilter.kt:42 and is called from three places") is fine and useful. A re-paste of the grep output isn't. The user (and future turns) can scroll back to the raw tool result if needed.

## What's deliberately outside this skill

These help efficiency too, but they aren't per-turn agent behaviors — they're either user actions or one-time setup, so they live elsewhere:

- **`/clear` at task boundaries** — only the user can run it. Useful when starting an unrelated task in the same session.
- **Picking a smaller model for routine work** — user-side (`/model`). Smaller tiers cost a fraction of the top tier per token; the exact ratio changes with each model generation, so check current pricing rather than assuming one. Best done at the start of a task: prompt caches are per-model, so switching mid-session re-processes the whole conversation uncached once.
- **Trim CLAUDE.md and MEMORY.md** — one-time edits to files that load on every session. The single highest-leverage action available, since the saving is paid forever.
- **Pre-allow safe commands** — use the `fewer-permission-prompts` skill to write an allowlist for read-only Bash commands, cutting approval round-trips.

If the user asks how to reduce token usage and these would help, suggest them — but they aren't tactics this skill applies on its own.

## Why this matters

Each turn sends the whole conversation again. Prompt caching makes the part that hasn't changed cheap to re-read, but not free, and anything new is written to the cache at a premium first. Appending to the conversation doesn't break the cache — the cache is a prefix match, so it only breaks when something *earlier* changes, the model changes, or the session sits idle past the cache lifetime. So the tactics here are not about protecting the cache; they're about *not adding* things that will be re-read forever.

That is why small habits beat one-shot optimizations. A 2,000-line command output pulled in at turn 5 of a 50-turn session isn't paid for once — it's paid for on each of the 45 turns that follow, and it brings the session closer to the point where context gets compacted and detail is lost. The cost of an unnecessary token is its size times the number of turns still to come, which makes the early turns of a long session the most valuable place to be careful.
