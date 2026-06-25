# claude-token-skills

Two small [Claude Code](https://claude.com/claude-code) skills that cut token usage — and the money — out of long coding sessions, without changing the answers you get.

- **`context-efficiency`** — per-turn habits that keep the working context small: pin bulk command output, don't re-read files, edit instead of rewrite, parallelize tool calls, don't echo tool output back into the conversation.
- **`delegate-reading`** — keep raw bulk content out of the thinking context on multi-file work: search before reading, read narrowly, and hand whole-file/whole-repo reading to a throwaway sub-agent so only the digest comes back.

The principle behind both: **every token in context gets reprocessed on every later turn, so trimming it compounds.** A 10% saving per turn is a 10% saving on *every* future turn too.

## Does it actually help? (measured, not vibes)

I ran a controlled A/B with Claude orchestrating the whole thing — generating the test repos, running every trial as a real headless `claude -p` session, and token-counting the transcripts. **38 runs**, same task / model / repo in both arms; the *only* variable was whether these two skills were loaded. Every run was checked for a correct answer before its tokens were counted.

| What was measured | Skills off | Skills on | Change |
|---|--:|--:|--:|
| Cost of a 12-step review (24k-line repo, when delegation engaged) | $1.76 | $0.80 | **−55%** |
| Working context for that same session | ~170k tok | ~87k tok | **−48%** |
| Worst-case run — "fix a bug in a 573-line file" | 5,804 tok | 346 tok | **−94%** |
| Worst-case run — "summarize a module" | 14,681 tok | 2,539 tok | **−83%** |
| Multi-step review session length | 14 turns | 8 turns | **−43%** |
| Bulk reading delegated to a sub-agent | 0% of runs | 100% of runs* | — |

\* On vague "survey the repo" prompts, delegation fired reliably. When a prompt names a *specific* file, the model still tends to just open it — so the savings land hardest on exploration, audits, and multi-file work. Honest caveat: the ceiling is ~50%; your mileage depends on how often delegation kicks in.

**The real win isn't a smaller average — it's that the expensive blow-ups stop happening.** Without the skills, the same request is a coin flip: sometimes lean, sometimes it reads an entire file for no reason. The skills cap the worst case.

## Install

Skills live in `~/.claude/skills/`. Copy both folders in:

```bash
git clone https://github.com/faizanOckto/claude-token-skills
cp -r claude-token-skills/skills/* ~/.claude/skills/
```

Or grab them without cloning the whole repo:

```bash
mkdir -p ~/.claude/skills
for s in context-efficiency delegate-reading; do
  mkdir -p ~/.claude/skills/$s
  curl -fsSL "https://raw.githubusercontent.com/faizanOckto/claude-token-skills/main/skills/$s/SKILL.md" \
    -o ~/.claude/skills/$s/SKILL.md
done
```

Restart Claude Code (or start a new session). The skills auto-trigger on multi-step coding work — debugging, refactors, investigations, "where is X", repo surveys, audits. You can also confirm they're loaded with `/context-efficiency` and `/delegate-reading`.

## How they work together

`context-efficiency` is the always-on per-turn discipline; `delegate-reading` is the heavy-hitter for the single most expensive thing an agent does — bulk reading. Loading both gives you the per-turn savings *and* the big structural win of never dragging raw source into the context that has to keep thinking.

## License

MIT — see [LICENSE](LICENSE). Use them, fork them, tweak the triggering, make them yours.
