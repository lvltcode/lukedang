# Multi-Agent Development Workflow: Practical AI Orchestration for Production Software

## The problem

Solo builders using AI coding tools hit a ceiling fast. A single model in a single context window can scaffold a feature, but production software requires decisions that span data modeling, security, UX, deployment, and business logic — often simultaneously. Feed all of that into one conversation and context degrades. The model forgets constraints from 40 messages ago. Bugs compound. You spend more time recovering than building.

The common response is to run multiple AI agents in parallel. This is worse. Two agents editing the same repo produce merge conflicts, ghost state, and silent regressions that cost hours to untangle. I lost 4–5 hours in a single session recovering from a dirty worktree I didn't catch before a handoff. That was enough to formalize the discipline.

The goal wasn't a rigid multi-agent system. It was a practical way to use different AI models for different types of work without the build falling apart.

## How it actually works

The core pattern is simple: use the right model for the right task, always sequentially, never in parallel on the same repo.

**Planning → Claude.** Requirements, data models, RLS policies, implementation sequencing. For larger systems like Cadence (24-table schema, 3 user roles), this produces a full document set before any code is written — product brief, implementation plan, component spec, data model, scope boundaries. For smaller tasks, it might be a single scoping conversation.

**Execution → Claude Code.** The primary builder. Multi-file refactoring, RLS implementation, edge functions, architectural changes. Git commit and clean worktree before every handoff. This is the non-negotiable discipline in the entire workflow.

**Overflow → Codex.** Isolated logic tasks, small fixes, refactors. Handles the work that doesn't need full codebase awareness, or picks up when Claude Code hits rate limits.

**Debate → ChatGPT (when needed).** Not every decision needs a second opinion. But for irreversible choices — data model design, authentication architecture, integration patterns, pricing logic — I'll open a separate conversation and stress-test the decision before committing. The filter: if you can undo it in five minutes, skip the debate.

**Orchestration → me.** I route tasks to the right model, check the worktree between handoffs, maintain the planning docs, and make the final call. This is the single point of failure. There's no automation layer. The judgment about which model to use for what, and when to skip a step, is the actual skill.

The workflow adapts per task. A complex multi-table feature gets the full treatment — planning docs, debate on the data model, sequential execution with checkpoints. A straightforward bug fix goes straight to Claude Code. Knowing which level of structure a task needs is more important than applying the same process to everything.

## What I learned

### Planning artifacts are the real memory

Chat history is disposable. PRD and markdown files are not. They are the only thing that compounds across ventures, across models, and across sessions. When I switch from Claude Code to Codex, or from one project to another, the planning documents carry the full context. The chat window doesn't.

Investment in planning artifacts has the highest ROI in the entire stack. Nothing else comes close. But the depth of planning scales with the complexity of the build — not every task needs five documents.

### Sequential over parallel

Running two agents on the same repo in parallel feels faster and produces ghost state — invisible conflicts between what each agent thinks the codebase looks like. One missed worktree check cost me 4–5 hours: half that time recovering what was salvageable, the other half rebuilding from scratch.

The rule: git commit and confirm a clean worktree before every model handoff. This is a $0 quality gate. It adds seconds per handoff and prevents the most expensive failure mode in the workflow.

### Context degrades at every model boundary

Every time a prompt moves from one model to another, the "why" behind a decision compresses into "what." The advisor knows the full reasoning. The executor receives a task. When the executor hits an edge case, it has no root reasoning to fall back on — so it guesses, or it asks, and either way the build stalls.

For complex decisions, I embed the reasoning directly into the planning docs using a simple structure: what was decided, why, and what constraints the executor must not violate. This isn't a rigid template applied to everything — it's a mitigation for the specific moments where context loss would cause real damage.

### Debate is expensive — use it selectively

Early on I spent cycles running UI tweaks and copy changes through a second model for validation. That was latency, not signal. The debate layer exists for decisions where being wrong means days of rework: data model changes, auth architecture, integration design. Everything else can be iterated in minutes.

## Trade-offs

This workflow has real costs. Context switching between models requires sustained attention — the orchestrator can't step away mid-session. As the sole decision-maker and router, I am the bottleneck. And simple tasks don't need this level of structure; applying the full workflow to a bug fix adds overhead that a single tool could handle faster.

These are conscious trade-offs. The alternative — one model, one context, no discipline — produces faster first drafts and worse production software.

## What compounds, what gets thrown away

The hardest lesson from running this across multiple ventures: most of what you build around AI tools is disposable. Model versions change. Prompt patterns that work today break next quarter. Tool configurations are temporary.

What compounds: planning artifacts, decision logs, scope discipline, and the judgment about when to apply structure and when to skip it.

What gets thrown away: specific prompt templates, model configurations, tool-specific setups. Anthropic ships faster than any solo builder can adapt. Alignment with their direction matters more than defending a custom workflow that worked last month.

## Current status

This workflow is in active use across multiple product builds including Cadence (music education platform) and three venture studio projects. The pattern maps to Anthropic's Advisor Strategy (released April 2026) — advisor model for planning, executor model for implementation, shared context — which I had been running manually before the official release.

The system is stable but never final. That's the point: discipline and judgment are portable, specific tool configurations are not.

---

*Built with: Claude (Opus), Claude Code, ChatGPT, Codex, Git, Supabase, Vercel, Playwright*

*Luke Dang — AI Product Builder · [GitHub](https://github.com/lvltcode) · [LinkedIn](https://www.linkedin.com/in/dangtranlevu/)*
