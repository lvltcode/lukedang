# Puppy Program OS: Building an Automated Foster Management Platform

[→ View the live prototype](https://puppy-prototype.vercel.app/)

## The problem

A national guide dog organization in Canada runs a foster program where volunteer families raise puppies for 12–18 months before the dogs enter formal training. Coordinators manage dozens of foster families simultaneously, tracking puppy development, behavioral patterns, health events, and socialization milestones — mostly through email chains, PDF manuals, and manual check-ins.

The coordination overhead is significant. Fosters forget to report issues. Staff spend hours chasing updates. Behavioral problems go undetected for weeks because there's no structured data flowing from foster homes back to the organization. By the time a pattern surfaces, the intervention window has narrowed.

The project brief: replace the static manual and email workflow with a structured digital system — daily logging, automated behavioral alerts, contextual learning content, and a staff dashboard that surfaces what needs attention without anyone having to ask.

## Users and workflow

Two distinct apps serve two roles:

**Foster app (mobile-first):** Volunteer families log daily observations — potty training, feeding, behavior, health, and socialization — through a chip-based input interface designed to complete in under 90 seconds. No typing, no forms. Tap the chips that apply, hit save. Content modules unlock progressively based on the puppy's age and days since placement, replacing the static PDF manual. Support tickets replace email.

**Staff app (desktop-first):** Coordinators open their day to an auto-populated dashboard showing which puppies need attention, ranked by urgency. Alert badges, support queue counters, and foster compliance indicators are all live from the database. Staff never need to ask "how is the puppy doing" — the data tells them before the foster does.

The core loop: Foster saves daily log → Postgres triggers evaluate behavioral rules → alerts auto-generate if patterns match → staff see alerts on dashboard → staff intervene with protocol checklists → foster acknowledges and acts.

## Architecture decisions

**Stack:** React + TypeScript frontend, Supabase backend (PostgreSQL + RLS + Auth), deployed on Vercel.

**18 tables with RLS from day one.** Every table carries an `org_id` column. Multi-tenancy is structural — a second organization can onboard without schema changes. Row-Level Security ensures organizations can never see each other's data, enforced at the database level.

**Alert engine lives entirely in Postgres.** This was the most consequential architecture decision. Four behavioral rules run as Postgres triggers — not in application code. Every time a daily log is saved, `fn_evaluate_all_rules` checks four conditions:

| Rule | Condition | Severity |
|---|---|---|
| Missing socialization | No socialization logged for 8+ consecutive days (weeks 8–20) | Medium → High at 14 days |
| Jumping pattern | Jumping on 3 of last 5 log days | Medium |
| Potty regression | 2+ accidents/day for 3 consecutive days (age >18 weeks) | Medium |
| Health flag | Vomiting or diarrhea on 2 consecutive days | High (immediate) |

Why Postgres triggers instead of Edge Functions or app logic? Three reasons: triggers can't be bypassed by a frontend bug, they execute atomically with the log save (no race conditions), and the alert deduplication logic uses a partial unique index that guarantees one open alert per puppy per rule type. A malformed API call or UI bug can't create duplicate alerts — the database won't allow it.

**Append-only audit trail.** Every alert status change (Open → In Review → Resolved) is recorded in `alert_events` with INSERT-only permissions. No UPDATE, no DELETE from the app layer. Full audit history for every intervention.

**Auth is magic-link only.** No passwords stored, no password reset flows, no credential leak surface. Fosters sign in with an email link. This matters because the foster population is non-technical — password management would generate support tickets.

**Data model core:**

```
organizations
    ↓
profiles (foster / staff / vet / org_admin)
    ↓
puppies → daily_logs → fn_evaluate_all_rules (trigger)
    ↓                         ↓
content_modules           alerts → alert_events (append-only)
(46 seeded, rule-based     ↓
 unlock by age/stage)   support_tickets
    ↓
foster_stats + foster_achievements (gamification)
```

**Key tradeoff — scope discipline:** The CBARQ-style behavioral assessment form (a standardized 100+ question instrument used in canine behavior research) was scoped and documented but deferred to post-pilot. It would have added significant complexity to the daily logging flow and wasn't needed to validate the core alert engine. Gamification (streaks, points, badges) was designed but deprioritized — nice-to-have, not launch-blocking.

## What makes this project technically interesting

**The 90-second constraint shaped everything.** If the daily log takes longer than 90 seconds, foster compliance drops and the alert engine starves for data. The entire input design — chip-based taps instead of text fields, six sections with sensible defaults, one-thumb mobile layout — exists to protect that completion time. UX decisions are data pipeline decisions.

**Rule-based content unlocking.** 46 content modules are seeded in the database with unlock conditions: `stage`, `day_after_placement`, `age_days`, and sex-specific filtering. Fosters never see a wall of content — modules appear when relevant. Staff can add new modules without a developer.

**Security posture for a non-profit context.** PIPEDA-compliant data minimization: no medical records, no SIN, no financial data. Only name, email, and optional phone. React Query cache keys include `orgId` to prevent cross-org data bleed in shared browser sessions. Cache cleared on sign-out.

## How AI was used in the build

**Planning phase:** Five structured planning documents (product brief, implementation plan, component spec, data model, scope boundaries) were generated through Claude before any code was written. The Postgres trigger logic — including the four alert rules, deduplication strategy, and escalation conditions — was designed entirely in conversation before being implemented.

**Build phase:** Lovable generated the initial React component library and page layouts. Claude Code handled the multi-file refactoring, RLS policy implementation, and trigger function logic. The alert engine required precise SQL that understood the full schema context — this is where Claude Code's multi-file reasoning was critical.

**Domain knowledge mattered.** I raised a puppy (Zuri) through the organization's foster program. That firsthand experience shaped the daily log structure, the alert thresholds, and the content unlock timing in ways that pure product research wouldn't have surfaced. Knowing that fosters check the app while holding a leash with one hand informed the 90-second, one-thumb design constraint.

## Current status

The prototype is live and was presented to program coordinators at the partner organization. The system was designed for a structured 1–2 month pilot with real foster workflows.

---

*Built with: React, TypeScript, Supabase (PostgreSQL + RLS + Triggers), Vercel, GitHub, Claude Code, Codex, Lovable*

*Luke Dang — AI Product Builder · [GitHub](https://github.com/lvltcode/lukedang) · [LinkedIn](https://www.linkedin.com/in/dangtranlevu/)*
