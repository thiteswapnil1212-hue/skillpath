# SkillPath

**Your Skills. Your Path. Your Progress.**

SkillPath is an adaptive coding-learning workspace for students and teachers. It was built as a team hackathon project for PS2 — Adaptive Learning and Assessment Portal.

Live demo (team deployment): https://devkraft-black.vercel.app/

The workspace shows only saved learner and teacher activity. Empty states are used until real activity is recorded — no sample learner, class, assignment, or activity records are created automatically.

## What it does

**For students**

- A concept map and learning pages that show what to learn next.
- A student dashboard with saved concept estimates and a deterministic next-mission recommendation when progress exists.
- A coding arena with a Monaco editor, session replay, and an explanation viva after passing hidden tests.
- Assessments with server-served questions and server-validated scoring, tracked per concept with Bayesian Knowledge Tracing (BKT) mastery.

**For teachers**

- An assignment workspace with saved drafts, review, publishing, and honest empty analytics states.
- Question generation from pasted notes or uploaded TXT, Markdown, PDF, or PPTX material, saved as drafts for review before publishing.
- A review queue that flags arena attempts with integrity signals for human review.

## How it works

- **Adaptive learning:** concept estimates are updated from assessment and arena results, and the next mission is recommended deterministically from saved progress.
- **Coding arena:** TypeScript, JavaScript, and Python starter snippets. Python runs locally in a pinned Pyodide Web Worker. Remote execution through JDoodle or Judge0 is optional and disabled unless explicitly configured with server-side credentials.
- **Mentor and viva:** local concept-note keyword retrieval and deterministic mentor heuristics. Passing hidden tests reveals an explanation viva generated from the submitted identifiers; heuristic answers update mastery without sending source to an external API. No generated explanation is connected by default.
- **RAG (first version):** keyword retrieval over teacher material, not embeddings or a vector database. Fine-tuning is intentionally not used.

## Tech stack

Next.js 16, React 19, TypeScript, Tailwind CSS, Monaco Editor, Pyodide, Supabase (optional), Vitest. See `package.json` for the full list.

## What works

- Responsive landing page with student, teacher, assessment, and coding arena navigation.
- Student dashboard with saved concept estimates and a deterministic next-mission recommendation when progress exists.
- Teacher assignment workspace with saved drafts, review, publishing, and honest empty analytics states.
- Server-served assessment questions and server-validated scoring with per-concept BKT mastery saved for an anonymous browser learner.
- Monaco code editor with TypeScript, JavaScript, and Python starter snippets. The arena can run visible samples and five hidden `countDown` cases through an explicitly enabled JDoodle provider; submissions update mastery.
- Arena attempts keep an event timeline in browser local storage and show a replay scrubber with integrity signals explicitly flagged for human review.
- Passing the arena hidden tests reveals an explanation viva generated from the submitted identifiers; heuristic answers update mastery without sending source to an external API.
- Local concept-note keyword retrieval and separate deterministic mentor heuristics. No generated explanation is connected.
- Typed domain and API contracts, Supabase SSR/browser client helpers, and adaptive-learning unit tests.

Local development can run without Supabase. Assessment mastery is saved to `.data/learner-mastery/` only after a local learner completes activity; production requires Supabase and never treats that local directory as durable storage.

## Start locally

```sh
npm install
cp .env.example .env.local
npm run dev
```

Then open `http://localhost:3000`. On Windows PowerShell, use `Copy-Item .env.example .env.local` instead of `cp`.

## Quality checks

```sh
npm run lint
npm run typecheck
npm test
npm run build
```

## Integrations and planned work

- Supabase credentials are optional. When configured, assessment mastery uses the `learner_mastery` table defined in `supabase/migrations/20261016000000_create_learner_mastery.sql`; otherwise the server writes only user-created records under the ignored `.data/` directory. The anonymous learner cookie is a local development identity, not authentication.
- JDoodle execution is disabled unless `JDOODLE_EXECUTION_ENABLED=true` and server-side credentials are present. This opts the public demo routes into potentially billable code execution; add authentication and deployment-backed rate/quota controls before enabling it on an internet-facing deployment. Judge0 remains optional and must be verified before use.
- Teacher PDF/TXT/Markdown/CSV uploads use server-only Gemini document understanding plus keyword RAG. Generated questions are Zod-validated, source-linked, saved as drafts, and publishable only after review. Supabase deployments need `GEMINI_API_KEY` and the server-only `SUPABASE_SERVICE_ROLE_KEY` for authoritative grading; without Supabase, the same workflow persists to ignored `.data/teacher-portal/state.json` for local development.
- Teacher notes can be pasted or uploaded as TXT, Markdown, PDF, or PPTX; every source uses the grounded Gemini route. Published assignments can also carry a validated visual explanation that appears as a diagram on `/learn`.
- RAG is keyword retrieval in this first version, not embeddings or a vector database. Fine-tuning is intentionally not used: teacher material changes per course and retrieval provides the grounding boundary.
- Session replay, code viva, persistent analytics, and validated adaptive-model parameters remain future work.

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md), [docs/API_CONTRACTS.md](docs/API_CONTRACTS.md), [docs/SETUP.md](docs/SETUP.md), and [docs/TEAM_WORKFLOW.md](docs/TEAM_WORKFLOW.md).

## Project structure

```text
src/app/            Next.js pages and API routes (student, teacher, arena, assessment)
src/components/     Landing, learning, coding, student, and teacher UI
src/features/      Adaptive learning (BKT, concept graph, missions) and assessments
src/data/          Concept notes and demo data
supabase/          SQL migrations for durable storage
docs/              Architecture, API contracts, setup, and team workflow
```

## Team

Built by a team. Initial ownership areas, from `docs/TEAM_WORKFLOW.md`:

- Aman Shejul — frontend and integration
- Swapnil Thite — backend and execution
- Pankaj Dandekar — assessments, adaptive missions, and QA
- Yash Ambekar — AI mentor, RAG, and viva

This repository is an imported copy of the original team repository, [AmanShejul/devkraft](https://github.com/AmanShejul/devkraft). Commit history is preserved from the original.
