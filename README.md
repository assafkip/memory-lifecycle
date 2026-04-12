# memory-lifecycle

Memory freshness, auto-archive, and decay promotion for Claude Code.

Every other memory tool helps Claude remember. This one tells Claude when to stop trusting what it remembers.

## The problem

Claude Code's auto-memory saves facts across sessions but treats every memory as equally valid forever. A "demo is in progress" memory persists after the demo dies. A "wait until Apr 14" memory persists after Apr 14 passes. Claude acts confidently on dead information.

## What this plugin does

Tags memories with temporal trust (`decay: fast | medium | slow`), warns at session start when fast-decay memories need verification, auto-archives stale memories, and promotes verified memories to slower decay tiers.

**Phase 1 (current):** All deterministic. No LLM in the loop. Pure Python, zero dependencies.

## Install

```bash
claude plugin add /path/to/memory-lifecycle
```

Or from GitHub:
```bash
claude plugin add assafkip/memory-lifecycle
```

## How it works

### Decay tagging

Add a `decay` field to memory file frontmatter:

```yaml
---
name: Josh demo engagement
type: project
decay: fast
---
Josh is reviewing the demo this week. No follow-up needed until Apr 14.
```

Three tiers:
- **fast** - facts that flip day-to-day (demos, deadlines, relationship stages). Must verify before acting.
- **medium** - facts that drift over weeks (IDs, team roles, configs). Should verify if acting on it.
- **slow** (default) - stable facts (voice rules, process conventions). Trust unless contradicted.

### Session start warnings

On every session start, the hook scans memory files and prints warnings:

```
MEMORY FRESHNESS WARNING
==================================================
These memories are marked decay: fast.
MUST verify before acting on their content.

  [FAST] Josh demo engagement (project_josh-demo.md)
  [FAST] Deadline for Antler IC (project_antler-deadline.md)
==================================================
```

### Verification tracking

After verifying a memory is still true, Claude updates the frontmatter:

```yaml
last_verified: 2026-04-11
verify_count: 3
```

This resets the staleness clock and contributes to promotion.

### Auto-archive

Stale memories get archived automatically on session start:
- `fast` decay: archived after 14 days without verification or update
- `medium` decay: archived after 60 days
- `slow` decay: never

Archived files move to `memory/archived/`. Reversible.

### Decay promotion

Verified memories get promoted to slower decay:
- `fast` with 3+ verifications and 30+ days old -> `medium`
- `medium` with 5+ verifications and 60+ days old -> `slow`

A fact that has been true for a month and verified three times is no longer fast-decay.

## File structure

```
memory-lifecycle/
  plugin.json              # Claude Code plugin manifest
  hooks/
    session-start.py       # Freshness + archive + promotion hook
  rules/
    memory-freshness.md    # Verification gate rules
  tests/
    test_lifecycle.py      # 32 tests covering all capabilities
  README.md
```

## Design decisions

- **No SQLite.** Frontmatter IS the database. Memories are small files. File I/O is fast enough.
- **No background worker.** claude-mem's top bugs are zombie processes, port collisions, and 110-second stop-hook blocks. We avoid all of that.
- **No LLM in the loop.** Every Phase 1 capability is deterministic. Date math, not prompt engineering.
- **Archive, don't delete.** Safe forgetting. Every archived memory can be restored by moving it back.
- **Exit 0 always.** The hook never blocks session start. It prints warnings, not errors.

## Tests

```bash
python3 tests/test_lifecycle.py
```

## Roadmap (Phase 2)

- Memory compilation: cluster 3+ related fast-decay memories into one slow-decay summary (LLM-assisted, user-invoked)
- Pitfall detection: parse transcripts for "that's outdated" corrections, auto-create fast-decay memories
- Marketplace submission

## License

MIT
