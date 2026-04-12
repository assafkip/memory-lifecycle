# memory-lifecycle

Every memory tool helps Claude remember things. This one tells Claude when to stop trusting what it remembers.

## What this solves

Claude Code remembers facts between sessions. That's useful. The problem is it treats every fact as equally true forever.

You tell Claude "Josh is reviewing the demo this week, don't follow up until April 14." Claude saves that. April 20 rolls around. Claude still thinks Josh is mid-review and holds off on the follow-up. Six days of silence because Claude trusted a stale fact.

This plugin adds an expiration layer to Claude's memory.

## How it works

You tag each memory with how fast it goes stale.

**fast** - changes day to day. Active demos, deadlines, "wait until Thursday," relationship status.

**medium** - drifts over weeks. Team roles, config IDs, integration settings.

**slow** - stable. Your preferences, process rules, voice conventions. This is the default.

The tagging looks like this in the memory file:

```yaml
---
name: Josh demo engagement
type: project
decay: fast
---
Josh is reviewing the demo this week. No follow-up until Apr 14.
```

## What happens at every session start

The plugin checks all your memories and does three things:

**1. Warns about fast-decay memories.**

Claude sees this before doing anything:

```
MEMORY FRESHNESS WARNING
==================================================
These memories are marked decay: fast.
MUST verify before acting on their content.

  [FAST] Josh demo engagement (project_josh-demo.md)
  [FAST] Antler IC deadline (project_antler-deadline.md)
==================================================
```

Claude now checks whether Josh is still reviewing before deciding to hold off on follow-up.

**2. Archives stale memories automatically.**

- Fast-decay memories go to archive after 14 days without being checked or updated
- Medium-decay memories go after 60 days
- Slow-decay memories stay forever

Archived means moved to a subfolder. Not deleted. You can always pull them back.

**3. Promotes verified memories.**

When Claude checks a fast-decay memory and confirms it's still true, it records that check. After 3 verifications over 30+ days, the memory gets promoted to medium-decay. After 5 verifications over 60+ days, medium becomes slow.

A fact that stayed true for a month and got checked three times is no longer volatile. The system recognizes that automatically.

## Install

```bash
claude plugin add assafkip/memory-lifecycle
```

That's it. No config. No API keys. No database. The plugin hooks into Claude Code's session start and runs automatically.

## What's inside

Four files. 230 lines of Python. Zero dependencies.

- `hooks/session-start.py` - the hook that checks, archives, and promotes
- `rules/memory-freshness.md` - instructions that tell Claude how to act on decay tags
- `plugin.json` - Claude Code plugin manifest
- `tests/test_lifecycle.py` - 32 tests covering every capability

## Why it's built this way

**No database.** The memory files themselves are the database. The decay tag lives in the file's header. Nothing to sync, nothing to corrupt.

**No background process.** The biggest memory tool in the ecosystem (claude-mem, 48K stars) runs a persistent HTTP server. Its top bugs are zombie processes and 110-second hangs on session close. This plugin runs once at session start, prints output, exits. Done.

**No AI in the loop.** Every decision is date math. "Is this memory older than 14 days with no verification? Archive it." No prompts, no token costs, no hallucinated decisions about what to keep.

**Archive, don't delete.** Stale memories move to a subfolder. Nothing is lost. You can restore any archived memory by dragging the file back.

## Verification tracking

When Claude checks a fast-decay memory and confirms it's still accurate, it updates two fields:

```yaml
last_verified: 2026-04-11
verify_count: 3
```

This resets the staleness clock (no archive for another 14 days) and counts toward promotion. Claude does this automatically when following the plugin's rules.

## Roadmap

**Shipping now (Phase 1):** Decay tagging, freshness warnings, auto-archive, promotion. All deterministic.

**Next (Phase 2):**
- Memory compilation - when 3+ related fast-decay memories are all archived, compile them into one slow-decay summary. Example: five "Josh demo status" updates become one "Josh Flashpoint engagement history."
- Pitfall detection - when Claude acts on a memory and the user corrects it ("no, that changed"), auto-create a fast-decay memory recording the correction.

## License

MIT
