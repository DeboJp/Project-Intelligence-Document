# Project Intelligence — Setup, Structure, Concerns & Benefits

---

## What It Actually Does

Every time you send a message to your agent IDE, the AI does three things automatically:

1. Reads `PROJECT_CONTEXT.md` before touching anything
2. Does the work you asked for
3. Updates `PROJECT_CONTEXT.md` with whatever was decided, explored, learned, or changed — using the right update strategy per section (overwrite, preserve, or evolve)

That's the whole mechanism. One rule file instructs the behavior. One output file accumulates the knowledge. No commands, no plugins, no setup per session.

---

## What You Actually Set Up

Two files. That's it.

```
your-project/
├── PROJECT_CONTEXT.md          ← Created automatically on first message
└── .antigravity/
    └── rules.md                ← You put this here once, manually
```

Or for other IDEs:

```
Claude Code  →  CLAUDE.md (project root)
Cursor       →  .cursorrules (project root)
OpenHands    →  agent config / system prompt file
```

The rule file content is identical regardless of IDE. Only the filename and location change.

---

## Setup Process

### Step 1 — Drop in the rule file in

Copy `RULE_memory-manager.md` content into your agent IDE's rule/instruction file. In Antigravity, that's `.antigravity/rules.md`. Save it.

That's the only manual step you will ever do.

### Step 2 — Start a project conversation

Say anything. "I want to build a scheduling app" or "Let's start on the auth system." The IDE agent will:

- Detect that `PROJECT_CONTEXT.md` doesn't exist
- Create it using the full template
- Fill in what it can infer
- Ask you one question: what problem are you solving and who has it?
- Update the document with your answer

### Step 3 — Work normally

From here you never think about the memory system. Just work. Every conversation updates the document automatically. New project? Same rule file works — it creates a fresh `PROJECT_CONTEXT.md` per project root.

---

## Structure Overview

`PROJECT_CONTEXT.md` has 20 sections, but they fall into four roles:

**Identity sections** (1-3) — What is this, why does it exist, who is it for. Written once, updated rarely. The founding context.

**Architecture sections** (4-9) — NFRs, system design, tech stack, data model, API contracts, security model. Updated when things change, with notes on what changed and why.

**Decision and history sections** (10-15) — Design decisions (ADR format), engineering standards, assumption log, risk register, conversations & thinking, exploration journal. These are additive. Nothing is deleted. When decisions are reversed, the old entry stays with a reversal block. When explorations fail, the failure is documented.

**Operational sections** (16-20) — Known issues, lessons learned, open questions, roadmap, references. Issues are resolved in place, never deleted. Roadmap tracks what was abandoned and why.

The rule file tells the AI exactly which update strategy to use for each section so it doesn't just overwrite history every time.

---

## Concerns

### Context window consumption

This is the biggest real tradeoff. At 500-600 lines, `PROJECT_CONTEXT.md` is roughly 10,000-15,000 tokens. Every single request starts by reading that file. With most frontier models that's a non-trivial but manageable cost — roughly 1-3 cents per request depending on provider and model.

Over a long project this adds up. Over a short project it barely registers.

Mitigation strategies:
- The 600-line limit in the rule exists specifically to cap this cost
- The compression instruction (consolidate Conversations & Thinking, summarize resolved sections) keeps the file from growing unboundedly
- If cost becomes a concern, split into a lean "hot" file (current status, active decisions, constraints) read every time, and a full archive file read on demand. Many teams run this pattern for large projects.

### Model compliance

The AI doesn't always follow instructions perfectly. Occasionally it will:
- Skip the file update at the end of a task
- Update the wrong section
- Overwrite something it should have preserved

This is not a flaw in the concept — it's current reality with LLMs. For now, treat the document as "maintained with high fidelity, not perfect fidelity." Periodically review it and correct errors. It's still vastly better than no documentation.

As models improve at following complex instructions, compliance improves.

### File quality degradation

Without periodic consolidation runs, the file drifts — vague entries accumulate, sections become redundant, outdated info lingers. The rule includes compression instructions but you should also run a manual consolidation prompt every few weeks:

> "Read PROJECT_CONTEXT.md and the codebase. Compress and refactor the
> memory file — merge duplicates, rewrite vague entries as clear permanent
> knowledge, archive old Conversations entries into summaries. The file
> should be cleaner, not longer."

### Not a replacement for proper documentation

`PROJECT_CONTEXT.md` is a living project brain. It is not:
- API documentation (use OpenAPI / docstrings)
- A user-facing README
- A deployment runbook
- A test specification

It complements those things by capturing the reasoning and history that formal docs never include.

---

## Benefits

**Continuity across sessions**
Models have no memory between conversations. This file is that memory. You never re-explain your architecture, re-justify a decision, or re-describe your stack. The AI arrives informed.

**Captures the why, not just the what**
Code shows you what was built. This file shows you why it was built that way, what else was considered, and what failed. That context disappears without a deliberate system to capture it. This is the system.

**Exploration history**
When you tried WebSockets and they didn't work, when you pivoted from one database to another, when you explored an approach and abandoned it — all of it is preserved. Future you (or a future model) won't propose the same dead-end solution again in a new conversation session, and will hopefully think a better solution with the added context.

**Decision traceability**
Every architectural decision has context, reasoning, alternatives rejected, and consequences documented. When someone asks "why did we use JWT instead of sessions?" the answer is in the file, not lost in a Slack thread from six months ago.

**Instant onboarding**
Paste `PROJECT_CONTEXT.md` into a fresh conversation. The model has full context in seconds. New team member joins? Same thing. The file is designed to be the complete brief.

**Assumption and risk visibility** 
Most projects fail on assumptions nobody wrote down. The assumption log and risk register force those invisible beliefs into explicit, trackable records. When an assumption is invalidated, it's right there.

**Trickle-down benefits compound over time**
Early sessions build the foundation. Later sessions benefit from that foundation automatically. By session 50, the AI understands your project at a depth and the same file can help a new engineer assimilate with codebase and system quicker. The depth also compounds, each session makes the next one smarter.

---

## What to Do Right Now

1. Copy `RULE_memory-manager.md` content into your IDE's rule file
2. Open a project and say anything
3. Let the AI create `PROJECT_CONTEXT.md` automatically
4. Answer its one question
5. Work normally (Core idea is you back and forth with AI alot)

The only ongoing action: run a consolidation pass every few weeks to keep the file clean. One prompt, five minutes.