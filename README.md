# project-context

A rule file to drop into your agent IDE, agent creates and maintains a `PROJECT_CONTEXT.md` for every project, covering documentation for the full SDLC.

Isn't it annoying to forget why reasoning behind decisions, or even past explorations for a project 6 months down the line. This rule addition shows you what was built, why JWT was chosen over sessions, why that database migration went sideways, and why an entire approach got scrapped after two days. The context that used to live in chat windows and Slack threads will now be documented via this project.

This is a rule file you drop into your agent IDE. It makes the AI automatically create and maintain a `PROJECT_CONTEXT.md` for every project, covering the full SDLC: architecture, tech stack decisions with actual reasoning, NFRs, security model, risk register, assumption tracking, and a full exploration journal that keeps failed approaches documented rather than forgotten. It reads the file before every task and updates it after. No commands needed.

---

## How it works

The rule file tells the agent three things: read `PROJECT_CONTEXT.md` before starting any task, do the work, then update the document based on what happened. Different sections follow different rules. Current state gets updated in place. Decision records, explorations, and lessons learned are append-only. If a decision gets reversed, the original stays with a block explaining what changed and why. Failed explorations stay in the journal because they're often the most useful context to have later.

First message on a new project: the agent detects there's no `PROJECT_CONTEXT.md`, creates it from the built-in template, fills in what it can infer, and asks one question to seed the rest. After that it runs on its own.

If you're adding this to an existing project rather than starting fresh, send this once to seed it:

> "Read the codebase and create PROJECT_CONTEXT.md. Fill in what you can infer from the existing code and ask me anything you can't figure out."


---

## Setup

Each project gets its own copy of the rule file and its own `PROJECT_CONTEXT.md`. Some IDEs support global rules that apply across all projects, in which case you only do this once.

Copy the contents of `RULE_memory-manager.md` into your agent IDE's instruction file:

- Antigravity: `GEMINI.md` or copy .agents/rules/ file to directory
- Claude Code: `CLAUDE.md` or create similar structure(.claude/rules/CLAUDE.md)
- Other agent IDEs: Look into individual website on how to setup rules, but likely your system prompt file, workspace instructions, or equivalent

Check `SETUP.txt` for a deeper guide.

---

## What's in the repo

`RULE_memory-manager.md` is the only file you need. It contains the full behavioral instructions and the document template the agent uses on first run. Everyone's preference may not be the same, so you should look into it and personalize its behavior to your liking.

`addl\EXAMPLE_PROJECT_CONTEXT.md` is an AI generated, filled-out example showing what the document should hopefully look like after a few weeks on a real project, including a superseded architecture decision, a failed infrastructure exploration, and an invalidated assumption. Worth reading before you set it up so you know what to expect.

---

## Tradeoffs

The document runs around 500-600 lines when healthy. That's roughly 10-15k tokens the agent reads before every task. On most frontier models that's a cent or two per request, which is fine until you're doing high volume and the project is large. The line limit in the rule exists to keep this in check.

Occasionally the agent skips an update or puts something in the wrong section. It's not perfect. Review the document occasionally and run a consolidation prompt when it starts to drift. Something along the lines of the examplpe below should help:

> "Read PROJECT_CONTEXT.md and the codebase. Compress and refactor the memory file, merge duplicates, summarize older conversation entries, and rewrite vague notes as permanent knowledge."

---

## The test

A simple test to see if its working, is observing wheather `PROJECT_CONTEXT.md` gets created on first message and updates after tasks.

---

If a team is using this it'd be interesting to consolidate everyones `PROJECT_CONTEXT.md` into a master version and visualize findings and workflow, and maybe benefitial for internal review as well.