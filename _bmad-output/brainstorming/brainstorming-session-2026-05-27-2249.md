---
stepsCompleted: [1, 2]
inputDocuments: ['docs/README.md']
session_topic: 'Subtle, modular gym management + members app for a medium-size gym — admin-configurable feature toggles, restrained core, room for out-of-the-box ideas. Not bound by the existing README.'
session_goals: 'Generate 100+ ideas across (a) the truly-used feature set of a medium gym (not bloated), (b) what gets toggled on/off per gym so the product flexes, (c) member-side delight & habit, (d) admin-side speed & trust, (e) out-of-the-box differentiators worth stealing from other domains.'
selected_approach: 'ai-recommended'
techniques_used: ['First Principles Thinking', 'SCAMPER Method', 'Cross-Pollination']
ideas_generated: []
context_file: 'docs/README.md'
---

# Brainstorming Session Results

**Facilitator:** YasasWeerasinghe
**Date:** 2026-05-27

## Session Overview

**Topic:** Subtle gym management + members management app — refining what is already planned in `docs/README.md` and surfacing what is not yet captured.

**Goals:** Generate 100+ ideas that (a) add subtle polish & differentiation, (b) expose hidden friction or risks in the current plan, (c) explore edges the README does not yet cover (UX, member psychology, admin workflows, data, offline, notifications, growth).

### Context Guidance

Loaded from `docs/README.md` — a comprehensive planning document for a Next.js 14 + Prisma + Auth.js gym PWA with:

- **Two roles:** ADMIN (member CRUD, plan builder, assignments, analytics) and MEMBER (today's workout, set logging, history).
- **Plan model:** drag-and-drop builder, `daysOfWeek Int[]`, multiple concurrent active assignments allowed.
- **Timer system:** three Int fields (`defaultRestSeconds`, `restSeconds`, `workSeconds`) + client-side timer.
- **Tracking granularity:** per-set rows in `ExerciseLog` (not per-exercise aggregates).
- **PWA from day one** with offline session-logging queue.
- **Intentionally excluded for now:** payments, classes, trainer chat, equipment booking, body measurements, nutrition.

**Where this session should push hardest:**

- Things the README treats as solved but probably aren't (e.g., what "today's workout" means when a member trains on a non-scheduled day, plan edits while a session is in-flight, multi-plan day collisions).
- The "subtle" lens — micro-interactions, copy, empty states, recovery from mistakes, mobile-thumb ergonomics — none of which are in the README.
- Member psychology and habit formation (streaks, nudges, recovery from missed workouts) — also absent.
- Admin workflows beyond the happy path (bulk operations, mistakes, the "I assigned the wrong plan" moment).
- Data trust and integrity (what does adherence mean when a member logs partial sets, or logs late?).

### Session Setup

**Approach:** AI-Recommended Techniques

**Refined framing (from user, 2026-05-27):**

- Not bound by `docs/README.md`. Treat it as one possible shape, not the spec.
- Scope: a **medium-size gym** — features that actually get used day to day, not a kitchen sink.
- **Modular admin app:** admin can enable/disable features per gym so the product feels right-sized for each owner.
- Out-of-the-box ideas welcome.
- **Hard constraint — $0 stack:** entire app must run on free tiers until real usage forces an upgrade. Free-tier limits are themselves a design input (file storage caps, email sends/month, DB row caps, function invocations, cold starts). Every feature must be costed against the free-tier budget, not just engineered.

## Technique Selection

**Phase 1 — First Principles Thinking** *(deep)*
Strip the gym to what it actually is, then rebuild. What does a medium-gym owner *truly* do every week? What does a member *truly* need every visit? Surface the "must exist" core before stacking features.

- Why: prevents feature-bloat by re-grounding before generation.
- Expected outcome: a short list of irreducible needs that anchor everything else.

**Phase 2 — SCAMPER Method** *(structured)*
Apply seven lenses (Substitute, Combine, Adapt, Modify, Put-to-other-use, Eliminate, Reverse) to the core flows and the **module-toggle architecture itself**. Especially powerful for deciding what to **Combine** (so toggles aren't a mess) and what to **Eliminate** (so the default install isn't bloated).

- Why: the user's modular-toggle requirement maps directly to SCAMPER's combine/eliminate lenses.
- Expected outcome: a structured grid of features ranked by toggle behavior and default-on/default-off.

**Phase 3 — Cross-Pollination** *(creative)*
Steal patterns from outside the gym world — Notion (modular blocks), Linear (keyboard-first admin), Duolingo (streak psychology), Strava (social proof), Spotify Wrapped (annual recap), Calendly (booking friction), Things 3 (today vs upcoming), Headspace (recovery framing).

- Why: this is the "out-of-the-box" lever — most gym apps look the same; the differentiators come from outside the category.
- Expected outcome: 20–30 borrowed-and-adapted ideas with clear source attribution.

**Total estimated time:** ~60–90 min depending on depth.

