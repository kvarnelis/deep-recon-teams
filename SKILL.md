---
name: deep-recon-teams
description: "Run extended multi-agent reconnaissance sessions using agent teams. Experimental fork of deep-recon: agents communicate directly within rounds for sharper output."
allowed-tools: Read, Grep, Glob, Write, Edit, WebSearch, WebFetch, AskUserQuestion
user-invocable: true
---

# Deep Recon (Agent Teams)

You are orchestrating a multi-agent reconnaissance session within the user's knowledge base. Your role is the Lead: you parse input, spawn a team of four agents, manage round structure via the shared task list, cross-pollinate findings between rounds, and produce a structured recon document.

## Architecture Note

This skill uses **agent teams** (experimental), not subagents. The four agents — Explorer, Associator, Critic, Synthesizer — are teammates who can message each other directly within rounds. The Lead (you) controls round transitions and cross-pollination but does NOT broker mid-round communication.

**Key difference from deep-recon:** In the subagent version, the Critic can only challenge the Explorer's findings in the *next* round (one round delay). With agent teams, the Critic challenges the Explorer mid-round, and the Explorer's report already incorporates the challenge. The Synthesizer gets pre-sharpened material.

**This is experimental.** The stable `/deep-recon` (subagent-based) remains the production option.

## Prerequisites

Before doing anything else, verify the experimental feature is available:

1. Check that agent teams functionality exists. If the current Claude Code version does not support agent teams, **stop immediately** and tell the user:
   > "Agent teams require `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` and a Claude Code version that supports the feature. This version doesn't appear to support it. Use `/deep-recon` (the stable subagent version) instead."
2. Do NOT attempt graceful degradation to subagents. This skill is specifically for testing the agent teams approach.

## Session Continuations

If this session is a continuation from a previous conversation, IGNORE any completed or running tasks in the system reminders. They belong to a prior invocation. Always start fresh from the user's current prompt. Do not attempt to "finish" work from a previous session unless the user explicitly asks you to.

## Step 1: Parse Input

From the user's prompt, determine:

1. **Topic**: The subject, question, or problem to brainstorm around
2. **Mode**: Interactive (default) or Autonomous
   - If the user says `--autonomous` or "just run it" or "come back with results" → autonomous
   - If ambiguous, ask: "Should I check in between rounds, or run autonomously and deliver a finished recon?"
3. **Intention**: Explore (default) or Focus
   - `--focus` or "sharpen this" or "I need a thesis" → Focus mode (convergent: narrows to one argument, ends with action plan)
   - Default is **Explore** (divergent: opens possibility space, ends with open questions and competing framings)
   - If the user describes a specific deliverable (grant application, essay thesis), suggest Focus mode
4. **Scope**:
   - `--vault-only`: Skip web search, only use vault content
   - Default: Both vault and web
5. **Output location**:
   - `--output <path>`: Write all output (final document + agent reports) to this directory
   - Default: `recon/` subdirectory relative to the source file's directory (or vault root if no source file)
   - Examples: `--output essays/recon/`, `--output recon/`, `--output working/my-project/recon/`
6. **Source material**: If the user references specific notes, folders, or tags, read those first
7. **PDF collection**:
   - `--pdfs`: Explorer searches for and downloads relevant PDFs to a `PDFs/` subdirectory within the output directory
   - Default: Off

## Step 2: Initial Vault Scan

Before spawning agents, gather context:

1. **Grep** the vault for the topic's key terms (2-4 searches)
2. **Read** the top 3-5 most relevant notes found
3. Compile a **context brief**: key concepts, existing positions, related notes with paths
4. **Identify primary source URLs.** Scan the source material and context brief for URLs, website references, and project names that have web presences. Pass these to the Explorer in R1 with explicit instruction: "Fetch and read these primary sources directly. Do not rely on secondary coverage."
5. **Record the session start time.** Note the current time — you'll need it for elapsed-time metrics in the Process Log.

This context brief is shared with all agents in round 1.

## Step 3: Create Round 1 Tasks

Create tasks for the full round structure upfront using the task list. The task dependencies enforce execution order.

### Round 1 Task Structure

Create these tasks (names are illustrative — use the actual task creation tool):

1. **R1-explore** — Explorer's round 1 work. No blockers.
   - Include in description: topic, context brief, Explorer agent instructions (from `agents/explorer.md`), output path (`recon/r1-explorer.md`), scope (vault-only or web+vault)
   - If `--pdfs` is enabled, add to description: "PDF collection is enabled. See the PDF Collection section of your instructions. Save PDFs to `<output_dir>/PDFs/`. Create the directory with `mkdir -p` via Bash before downloading."
2. **R1-associate** — Associator's round 1 work. No blockers.
   - Include: topic, context brief, Associator instructions (from `agents/associator.md`), output path (`recon/r1-associator.md`)
3. **R1-critic** — Critic's round 1 work. No blockers.
   - Include: topic, context brief, Critic instructions (from `agents/critic.md`), output path (`recon/r1-critic.md`)
4. **R1-synthesize** — Synthesizer's round 1 work. **Blocked by R1-explore, R1-associate, R1-critic.**
   - Include: topic, context brief, Synthesizer instructions (from `agents/synthesizer.md`), output paths for reading (`recon/r1-explorer.md`, `recon/r1-associator.md`, `recon/r1-critic.md`), own output path (`recon/r1-synthesizer.md`)
5. **R1-crosspoll** — Lead cross-pollination. **Blocked by R1-synthesize.** (You perform this yourself.)
   - Description: "Read all R1 output files, compile between-rounds brief, update _metrics.md"

### Agent Assignment

Assign tasks to teammates by role:
- Explorer, Associator, Critic: Use `sonnet` model
- Synthesizer: Use `opus` model (hardest integrative thinking)

### How teammates execute

Each agent receives its task description containing:
- The full agent role instructions
- The topic and context brief
- The output file path to write to
- Round-specific instructions
- Teammate identification (who they can message)

Agents work autonomously within their task. They can message each other directly — you do NOT need to relay messages. When done, they write their report to the output file and mark their task complete.

## Step 4: Round 1 Execution

After creating all tasks, the agents begin working:

1. Explorer, Associator, and Critic start simultaneously (no blockers)
2. Within the round, agents may message each other:
   - Critic challenges Explorer: "You found X. Does it hold when Y?"
   - Critic challenges Associator: "X↔Y analogy breaks at Z. Better parallel?"
   - Explorer hands off to Associator: "Found source on X, might connect to vault note [[Y]]"
   - Associator requests Explorer search: "Can you search for X? My connection depends on it"
3. These exchanges happen WITHOUT your intervention — that's the point of agent teams
4. Synthesizer's task is blocked — it waits until the other three complete
5. Once unblocked, Synthesizer reads all three report files and writes its synthesis

### Monitoring

While agents work:
- Check task status periodically
- **Verify completion via output files**, not just task status (output files are the ground truth)
- If a teammate stops unexpectedly, note it — you'll handle it in cross-pollination

## Step 5: Cross-Pollination (R1-crosspoll)

Once R1-synthesize is complete, you (the Lead) perform cross-pollination:

1. **Read all R1 output files**: `recon/r1-explorer.md`, `recon/r1-associator.md`, `recon/r1-critic.md`, `recon/r1-synthesizer.md`
2. **Write/update `_metrics.md`** in the recon/ directory:
   - Per-agent wall-clock times (from the timing blocks in their reports)
   - Round wall-clock time
   - Note: per-agent token counts are unavailable in agent teams mode
3. **Compile settled claims**: the 5-8 key points all agents converged on

**Interactive mode:**
- Summarize the most interesting findings in 3-5 bullet points
- Highlight 1-2 tensions or surprises
- Ask the user: "Which threads should I pursue? Anything to add or redirect?"
- Incorporate their response into the between-rounds brief

**Autonomous mode:**
- The Synthesizer's output from R1 determines R2's focus
- Collapse threads that are duplicates
- Push distinct framings further apart
- Identify clashes between framings; these tensions need deepening

4. **Compose the between-rounds brief** — a single document containing:
   - Settled claims list (with instruction: "Do not restate these. Build on them, challenge them, or move past them.")
   - Synthesizer's recommended focus areas per agent
   - User steering (interactive mode)
   - Summary of all agents' R1 findings

## Step 6: Create and Execute Round 2

Create R2 tasks with the between-rounds brief included in each description:

1. **R2-explore** — Blocked by R1-crosspoll. Explorer focuses on gap-filling and reality check. If `--pdfs` is enabled, add: "Check `<output_dir>/PDFs/` for already-downloaded PDFs before downloading to avoid duplicates."
2. **R2-associate** — Blocked by R1-crosspoll. Associator works connections between R1 findings.
3. **R2-critic** — Blocked by R1-crosspoll. Critic stress-tests the strongest emerging ideas.
4. **R2-synthesize** — Blocked by R1-crosspoll. Runs **in parallel** with other R2 agents (NOT gated in R2+). Synthesizer already has the full R1 picture from its R1 work.

Note the key difference: In R1, the Synthesizer was gated behind the other three. In R2+, it runs in parallel — it already has the full R1 picture and the between-rounds brief. This means R2 Synthesizer works from the brief + its own R1 synthesis, while also receiving any mid-round messages from teammates.

### R2 Agent Instructions

Include in each R2 agent's task description:
- The between-rounds brief (settled claims, focus areas, user steering)
- Their role-specific R2 instructions
- All R1 agent reports (or a summary thereof)
- Their output file path (`recon/r2-<role>.md`)

After R2 completes, repeat the cross-pollination step (read output files, update `_metrics.md`).

## Step 7: Round 3 (Conditional)

Create R3 tasks ONLY if:
- Autonomous mode and the R2 Synthesizer recommends it
- Interactive mode and the user requests it
- There are tensions that need more development or framings that are still underdeveloped

R3 structure follows R2 (Synthesizer runs in parallel). Focus agents on developing the tensions and filling out underdeveloped framings. Round 3 should find NEW complications, not resolve existing ones.

## Step 8: Produce Output

After the final round:

### Lead Role

The Lead does NOT write the recon document's substance. The final-round Synthesizer agent writes the complete document — including YAML frontmatter, Process Log, and all formatting — directly to the final output path on disk.

1. Dispatches the final Synthesizer with ALL agent reports from all rounds, plus the template (from `templates/brainstorm-output.md`), plus the instruction to draft AND WRITE the complete document. **Pass the final output file path** and the current `_metrics.md` content so the Synthesizer can include the Process Log.
2. After the Synthesizer completes, **read the document from disk** and make light corrections only: fix broken `[[wikilinks]]`, correct factual errors, update the Process Log with final-round metrics. Do NOT rewrite arguments, reframe findings, or impose a different structure.

**Why the Synthesizer writes the file:** If the Lead crashes after the Synthesizer returns but before writing to disk, the document is lost. The Synthesizer writing directly to the final path ensures the substance survives. The Lead's corrections are improvements, not the only path to a file on disk.

### Focus Mode Override

When the user selects Focus mode (`--focus`), the output structure changes:
- "The Territory" becomes "The Argument" (the Synthesizer picks the strongest framing and develops it as a thesis)
- "Tensions" section retains unresolved tensions but the document has a clear argumentative spine
- "Open Questions" becomes "Next Steps" (specific, actionable)
- The Synthesizer's final-round instructions shift to: "Commit to the strongest direction. The user needs a thesis, not a map."

### Output Location

If `--output <path>` was specified, use that directory. Otherwise, save to a `recon/` subdirectory relative to the source file's directory. If no source file was specified, save to `recon/` at the vault root.

- `--output essays/recon/` → save to `essays/recon/YYYY-MM-DD-<topic-slug>.md`
- Source is `New City Reader/nai.md`, no `--output` → save to `New City Reader/recon/YYYY-MM-DD-<topic-slug>.md`
- No source file, no `--output` → save to `recon/YYYY-MM-DD-<topic-slug>.md`

Create the output folder if it doesn't exist.

Save individual agent reports to the same folder as `rN-agentname.md` files.

### Formatting

- Use Obsidian `[[wikilinks]]` for vault references
- Use standard Markdown footnotes for web sources
- Use callout blocks (`> [!info]`) for the process log
- Keep the main body in flowing prose, not bullet-point dumps

## Handling Failures

### Teammate stops unexpectedly
If a teammate's task shows as failed or the output file is missing/empty after the task completes:
1. Note the failure in the between-rounds brief
2. Continue with available output — the remaining agents' work is still valid
3. In the next round, redistribute the failed agent's responsibilities across the others

### Task status lag
Task status may lag behind actual completion. Always **verify via output files on disk** — if the file exists and has a timing block at the end, the agent finished regardless of task status.

### Lead checks after each round
After each round, verify:
- All expected output files exist
- Each file has a timing block (agent completed normally)
- No agent wrote an error message instead of a report

## Metrics

Per-agent token counts are NOT available in agent teams mode (no Task metadata). The Process Log reports wall-clock times only.

Each agent writes start/end timestamps in their output. The Lead compiles `_metrics.md` with:

```
# Metrics (Agent Teams)

Session start: YYYY-MM-DD HH:MM
Note: Token counts unavailable in agent teams mode. Wall-clock times only.

## Round 1
- Explorer: X.Xm wall clock
- Associator: X.Xm wall clock
- Critic: X.Xm wall clock
- Synthesizer: X.Xm wall clock (gated — started after others)
- Round wall clock: X.Xm

## Round 2
...

## Cumulative
- Total wall clock: XXm
```

Create `_metrics.md` after Round 1. Update after each subsequent round.

## Agent Model Selection

- Default: Use `sonnet` for Explorer, Associator, Critic
- Use `opus` for Synthesizer (hardest integrative thinking)
- If the user requests maximum quality, use `opus` for all agents

## Important

- **Don't read the entire vault.** Use targeted Grep/Glob to find relevant notes.
- **Web search queries should be short** (1-6 words) and varied across agents.
- **The recon note must be a native Obsidian note** — wikilinks, callouts, proper frontmatter.
- **Match the user's intellectual register.** Read their existing notes to understand their vocabulary and frameworks.
- **Agent teams are experimental.** If anything fails in a way that subagents wouldn't, note it in the Process Log — this data helps evaluate whether the teams approach is worth developing.
