# deep-recon-teams

**Experimental fork of [deep-recon](https://github.com/kvarnelis/deep-recon)** that uses Claude Code's agent teams instead of subagents.

Four specialized agents run in parallel rounds — but now they can talk to each other mid-round. The hypothesis: inter-agent dialogue (Critic challenging Explorer directly, Synthesizer requesting clarification) produces sharper output than brokering everything through the orchestrator.

## Architecture

### deep-recon (subagents) — hub-and-spoke

```
         ┌──────────┐
         │   Lead    │
         └────┬─────┘
        ┌─────┼─────┐
        ▼     ▼     ▼     ▼
     Explore  Assoc  Crit  Synth
        │     │     │     │
        └─────┼─────┘     │
              ▼            │
         Lead collects ────┘
         Lead redistributes
              ▼
         Next round
```

Agents report to the Lead. The Lead digests, cross-pollinates, dispatches the next round. Critic→Explorer challenges take a full round delay.

### deep-recon-teams (agent teams) — mesh within rounds

```
         ┌──────────┐
         │   Lead    │
         └────┬─────┘
              │ (round transitions only)
        ┌─────┼─────┐
        ▼     ▼     ▼     ▼
     Explore ↔ Assoc ↔ Crit → Synth
        │     │     │        │
        └─────┼─────┘        │
              │               │
         Lead reads files ────┘
         Lead cross-pollinates
              ▼
         Next round
```

Agents message each other within rounds. The Critic challenges the Explorer and Associator *during* the round. The Explorer's report already incorporates the challenge. The Synthesizer gets pre-sharpened material.

The Lead still controls round transitions and cross-pollination — the interpretive step between rounds stays deterministic.

### Round structure

**Round 1:**
- Explorer, Associator, Critic start simultaneously
- Mid-round messaging: Critic challenges, Explorer/Associator handoffs
- Synthesizer is **gated** — waits until the other three finish, then reads their reports
- Lead cross-pollinates after Synthesizer completes

**Round 2+:**
- All four agents start simultaneously (Synthesizer runs in parallel)
- Mid-round messaging continues
- Lead cross-pollinates after the round

**Round 3:** Created dynamically if the Synthesizer recommends it (autonomous) or the user requests it (interactive).

### Inter-agent messaging patterns

| Sender | Recipient | Pattern |
|---|---|---|
| Critic | Explorer | "You found X. Does it hold when Y?" |
| Critic | Associator | "X↔Y analogy breaks at Z. Better parallel?" |
| Explorer | Associator | "Found source on X, might connect to vault note [[Y]]" |
| Associator | Explorer | "Can you search for X? My connection depends on it" |
| Synthesizer | Any | "Your finding X contradicts Z. Clarify?" |

Messages are brief (2-3 sentences), actionable. No agent messages the Lead mid-round.

## The Agents

| Agent | Style | Role |
|---|---|---|
| **Explorer** | Divergent | Casts the widest net — web searches, vault searches, adjacent fields, historical parallels |
| **Associator** | Lateral | Finds non-obvious connections — structural analogies, cross-domain pattern matching, productive metaphors |
| **Critic** | Adversarial | Stress-tests emerging ideas — prior art, hidden assumptions, steelmanned objections, productive contradictions |
| **Synthesizer** | Integrative | Identifies emergent patterns across all inputs — distinct framings, tensions worth preserving, the question the user hadn't considered |

## Modes

| Flag | Mode | Description |
|---|---|---|
| *(default)* | Interactive | Socratic — checks in between rounds for user steering |
| `--autonomous` | Autonomous | Runs all rounds, delivers finished document |
| *(default)* | Explore | Divergent — opens possibility space, ends with open questions |
| `--focus` | Focus | Convergent — narrows to one argument, ends with action plan |
| `--vault-only` | Vault-only | Skips web search, uses only vault content |

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with agent teams support
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in environment or settings.json
- An Obsidian vault
- Web search access (unless using `--vault-only`)

The skill **fails fast** if agent teams aren't available. No graceful degradation — use `/deep-recon` instead.

## Installation

Copy the `deep-recon-teams` directory into your `.claude/skills/` directory:

```
your-project/
└── .claude/
    └── skills/
        └── deep-recon-teams/
            ├── SKILL.md
            ├── agents/
            │   ├── explorer.md
            │   ├── associator.md
            │   ├── critic.md
            │   └── synthesizer.md
            └── templates/
                └── brainstorm-output.md
```

Then add the skill to your `CLAUDE.md`:

```markdown
### /deep-recon-teams - Deep Reconnaissance (Agent Teams)
Experimental fork of deep-recon using agent teams for inter-agent dialogue.
Requires CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1. Use /deep-recon for stable version.
```

## Usage

```
/deep-recon-teams <topic or question>
```

Examples:
```
/deep-recon-teams the relationship between infrastructure decay and political legitimacy
/deep-recon-teams --autonomous --focus what makes network culture different from digital culture
/deep-recon-teams --vault-only connections between my notes on sound art and landscape
```

## Experimental Caveats

1. **Token cost**: Expect 3-5x more tokens than subagent deep-recon. Inter-agent messaging adds overhead. The Synthesizer gating in R1 adds a sequential step.
2. **No session resumption**: If the session breaks mid-run, output files on disk are the recovery path. Individual agent reports in `recon/rN-<role>.md` survive.
3. **Task status lag**: The Lead verifies completion via output files on disk, not just task status. If the file exists and has a timing block, the agent finished.
4. **Shutdown can be slow**: After the final round, the Lead proceeds with formatting while teammates wind down.
5. **No per-agent token counts**: Agent teams mode doesn't expose Task metadata. The Process Log reports wall-clock times only.
6. **Untested at scale**: This is an experiment to compare output quality between subagent and agent-teams architectures. Report issues to help evaluate the approach.

## Metrics

Per-agent token counts are unavailable in agent teams mode. The Process Log reports:
- Per-agent wall-clock times (from timestamps each agent writes)
- Per-round wall-clock times
- Cumulative wall-clock time

Compare with subagent deep-recon's token metrics to evaluate cost/quality tradeoff.

## Output

Same structure as deep-recon:

- **Central Question** — the refined driving question
- **The Territory** — 3-5 fully developed framings
- **Tensions** — productive frictions between framings
- **Unexpected Connections** — cross-domain links as prose
- **Open Questions** — genuinely open, not rhetorical
- **Sources** — vault wikilinks and web references
- **Process Log** — mode, intention, round count, wall-clock times

Individual agent reports are saved alongside as reference material.

## vs. deep-recon

| | deep-recon | deep-recon-teams |
|---|---|---|
| Architecture | Subagents (Task tool) | Agent teams (experimental) |
| Inter-agent comms | None (Lead brokers) | Direct mid-round messaging |
| Critic→Explorer delay | 1 round | Same round |
| Synthesizer R1 | Parallel with others | Gated (waits for others) |
| Token tracking | Per-agent counts | Wall-clock only |
| Token cost | Baseline | ~3-5x more |
| Stability | Production | Experimental |
| Env requirement | None | CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 |

## License

MIT
