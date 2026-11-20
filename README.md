# SkillPath

**Your Skills. Your Path. Your Progress.**

SkillPath is an adaptive coding-learning workspace for students and teachers. It was built as a team hackathon project for PS2 — Adaptive Learning and Assessment Portal.

**Live demo (team deployment):** https://devkraft-black.vercel.app/

The workspace shows only saved learner and teacher activity. Empty states are used until real activity is recorded — no sample learner, class, assignment, or activity records are created automatically. The landing page shows `0+` counters until real data exists.

## Screenshots

![SkillPath landing page](docs/screenshots/landing.png)

Landing page — dark hero 'Learn with direction.' with a spiral starfield; stats 6+ concepts mapped, 24+ focused missions, 3+ languages ready. Screenshot of the live demo, 20 Nov 2026.

## What it does

**For students**

- A concept map and learning pages that show what to learn next.
- A student dashboard with saved concept estimates and a deterministic next-mission recommendation when progress exists.
- A coding arena with a Monaco editor, session replay, and an explanation viva revealed on Submit (even if the hidden tests fail).
- Assessments with server-served questions and server-validated scoring, tracked per concept with Bayesian Knowledge Tracing (BKT) mastery.

**For teachers**

- An assignment workspace with saved drafts, review, publishing, and honest empty analytics states.
- Question generation from pasted notes or uploaded TXT, Markdown, PDF, or PPTX material, saved as drafts for review before publishing.
- A review queue that flags arena attempts with integrity signals for human review.

## How the core loop works

A student practices, the system updates what it believes they know, and the next mission follows from that — deterministically, not from a black-box model.

```mermaid
sequenceDiagram
    actor Student
    participant Dashboard as Student dashboard
    participant Arena as Coding arena (Monaco)
    participant Runner as Code runner
    participant Viva as Explanation viva
    participant BKT as BKT mastery
    participant Mentor as Mentor (keyword RAG)

    Student->>Dashboard: Open dashboard
    Dashboard->>BKT: Read saved concept estimates
    BKT-->>Dashboard: Deterministic next-mission recommendation
    Dashboard-->>Student: Show what to learn next
    Student->>Arena: Write a solution (TS / JS / Python)
    Arena->>Runner: Run visible samples
    Runner-->>Arena: Pass / fail
    Arena->>Runner: Run hidden tests (5 countDown cases)
    Runner-->>Arena: Hidden tests pass / fail
    Student->>Arena: Submit solution
    Arena->>Viva: Reveal explanation viva (on Submit, even if hidden tests fail)
    Viva->>Student: Questions generated from submitted identifiers
    Student->>Viva: Heuristic answers (no source sent to any external API)
    Viva->>BKT: Update mastery from viva + attempt
    BKT-->>Dashboard: New estimates, new next mission
    Student->>Mentor: Ask about a concept
    Mentor->>Mentor: Keyword retrieval over concept notes
    Mentor-->>Student: Deterministic heuristic answer
```

Notes on the diagram, all true to the implementation:

- **Python runs locally** in a pinned Pyodide Web Worker (`public/pyodide-worker.js`). Remote execution through JDoodle or Judge0 is optional and **disabled unless explicitly configured** with server-side credentials.
- **The viva is generated from the submitted identifiers** (function and variable names in the student's own code). Heuristic answers update mastery without sending source to an external API. **No generated explanation is connected by default.**
- **The viva is revealed on Submit, not on passing.** Per `docs/ARCHITECTURE.md`, the arena reveals viva questions on Submit regardless of hidden-test results or provider availability, so a learner can still explain a solution when billable execution is disabled. (Corrected 20 Nov 2026 — the earlier diagram implied the viva only appeared after the hidden tests passed.)
- **The mentor is local keyword retrieval** over concept notes plus deterministic heuristics — not embeddings, not a vector database, not an LLM call.
- **RAG (first version) is keyword retrieval** over teacher material. Fine-tuning is intentionally not used: teacher material changes per course, and retrieval provides the grounding boundary.

## How it works (data flow)

```mermaid
flowchart LR
    Student([Student]) --> App["Next.js App<br/>student · arena · assessment · learn"]
    Teacher([Teacher]) --> App
    App --> Features["src/features<br/>BKT mastery · concept graph · missions"]
    App --> Arena["Coding arena<br/>Monaco + Pyodide worker · hidden tests"]
    Arena --> Viva["Explanation viva<br/>revealed on Submit"]
    App --> Mentor["Mentor · keyword RAG<br/>concept notes · no embeddings"]
    Teacher --> Gen["Question generation<br/>Gemini doc understanding"]
    Gen --> Review["Review queue<br/>drafts → human review → publish"]
    App --> Local["Local storage<br/>arena event timeline + replay<br/>.data/ (local dev only)"]
    App --> DB["Supabase (optional)<br/>learner_mastery table"]
```

Where data lives:

- **Browser local storage** — arena attempt event timelines and the replay scrubber.
- **`.data/learner-mastery/`** — assessment mastery on local development only, written after a local learner completes activity. Production requires Supabase and never treats that directory as durable storage.
- **`.data/teacher-portal/state.json`** — teacher drafts on local development only, when Supabase is not configured.
- **Supabase `learner_mastery` table** — durable mastery storage when credentials are configured (migration: `supabase/migrations/20261016000000_create_learner_mastery.sql`).
- **Anonymous learner cookie** — a local-development identity, not authentication. There is no sign-in system beyond the `/login` page surface; do not treat the cookie as an account.

## Database schema (Supabase)

Supabase is optional: without credentials the server persists mastery and teacher state to the ignored `.data/` directory (local development only). When Supabase is configured, durable storage is 11 tables across 7 migrations in `supabase/migrations/`, all with row-level security enabled.

```mermaid
erDiagram
    auth_users {
        uuid id PK
    }
    learner_mastery {
        uuid learner_id PK "cookie UUID or auth.uid()"
        jsonb mastery
        timestamptz updated_at
    }
    learner_viva_attempts {
        uuid learner_id PK "cookie UUID or auth.uid()"
        text mission_id PK
        timestamptz counted_at
    }
    teacher_documents {
        uuid id PK
        uuid owner_id FK "auth.users"
        text name
        text mime_type
        text status "processing / ready / failed"
        text failure_reason
        timestamptz created_at
    }
    teacher_document_chunks {
        text id PK
        uuid document_id FK
        uuid owner_id FK "auth.users"
        text content
        int page_number
        text topic
        jsonb concepts
        timestamptz created_at
    }
    quiz_sets {
        uuid id PK
        uuid owner_id FK "auth.users"
        text title
        text subject
        text topic
        uuid topic_id
        text learning_stage "diagnostic / practice"
        text status "draft / reviewed / published / archived"
        jsonb visual_explanation
        timestamptz available_from
        timestamptz due_at
        timestamptz published_at
        timestamptz created_at
    }
    quiz_questions {
        text id PK
        uuid quiz_set_id FK
        uuid owner_id FK "auth.users"
        text question_type "mcq / short-answer"
        text prompt
        jsonb options
        text explanation
        text difficulty "foundation / intermediate / advanced"
        uuid source_document_id FK "teacher_documents, nullable"
        text generation_status "grounded / needs-review"
        text review_status "draft / reviewed / published"
        timestamptz updated_at
    }
    quiz_question_answers {
        text question_id PK "FK quiz_questions"
        uuid owner_id FK "auth.users"
        text correct_answer
    }
    quiz_attempts {
        uuid id PK
        uuid quiz_set_id FK
        uuid learner_id FK "auth.users"
        timestamptz submitted_at
        int score
        int total
        text learning_stage "diagnostic / practice"
    }
    quiz_responses {
        uuid attempt_id PK "FK quiz_attempts"
        text question_id PK "FK quiz_questions"
        text response
        boolean is_correct
        text topic
        text difficulty
        text concept_id
    }
    student_profiles {
        uuid id PK "FK auth.users"
        text email
        timestamptz created_at
        timestamptz last_activity_at
    }
    student_learning_events {
        uuid id PK
        uuid learner_id "auth.uid(), no FK"
        uuid quiz_set_id FK
        uuid topic_id
        text event_type "topic_opened / diagnostic_completed / explanation_viewed / concept_revisited / practice_completed / topic_completed"
        text concept_id
        timestamptz occurred_at
        jsonb metadata
    }

    auth_users ||--o{ teacher_documents : owns
    auth_users ||--o{ teacher_document_chunks : owns
    auth_users ||--o{ quiz_sets : owns
    auth_users ||--o{ quiz_questions : owns
    auth_users ||--o{ quiz_question_answers : owns
    auth_users ||--o{ quiz_attempts : takes
    auth_users ||--|| student_profiles : "is"
    teacher_documents ||--o{ teacher_document_chunks : "split into"
    teacher_documents ||--o{ quiz_questions : "grounds"
    quiz_sets ||--o{ quiz_questions : contains
    quiz_sets ||--o{ quiz_attempts : attempted
    quiz_sets ||--o{ student_learning_events : "events for"
    quiz_questions ||--|| quiz_question_answers : "answer key for"
    quiz_questions ||--o{ quiz_responses : answered
    quiz_attempts ||--o{ quiz_responses : contains
```

Notes on the schema, all read from the migrations:

- **Anonymous vs authenticated identity.** `learner_mastery.learner_id`, `learner_viva_attempts.learner_id`, and `student_learning_events.learner_id` are plain `uuid` columns with no foreign key: for anonymous learners they hold the UUID from the learner cookie (`x-learner-id` header); for signed-in learners they hold `auth.uid()`. Only `quiz_attempts.learner_id` and `student_profiles.id` are declared foreign keys to `auth.users(id)`.
- **Viva mastery is claimed once.** `learner_viva_attempts` has primary key `(learner_id, mission_id)` — one counted viva per learner per mission.
- **Answer keys are teacher-only.** `quiz_question_answers` holds `correct_answer` per question; students never receive it — the question endpoint returns only prompts and choices, and RLS restricts the answers table to the owning teacher.
- **Grounding is traceable.** Every `quiz_questions` row records its source document, page, and excerpt (`source_document_id`, `source_page`, `source_excerpt`) plus `generation_status` (`grounded` / `needs-review`).
- **Learning stages.** `quiz_sets` and `quiz_attempts` carry `learning_stage` (`diagnostic` / `practice`); `student_learning_events` records the learner journey (`topic_opened`, `diagnostic_completed`, `explanation_viewed`, `concept_revisited`, `practice_completed`, `topic_completed`).
- **Original uploads are not kept.** Teacher documents store extracted chunks only; the `teacher_document_chunks` table comment states the original uploaded binaries are not retained.
- `auth_users` in the diagram is Supabase Auth's built-in `auth.users` table, not a migration in this repo.

## Tech stack

| Layer | Choice |
|---|---|
| Framework | Next.js 16.3.7 (App Router) · React 19.3.0 · TypeScript 5.9.3 |
| Styling | Tailwind CSS 3.4.17 · PostCSS · Autoprefixer |
| Editor | `@monaco-editor/react` 4.7.0 |
| Code execution | Pyodide Web Worker (local, pinned; Pyodide is not an npm dependency) · JDoodle / Judge0 optional, disabled by default |
| Data | `@supabase/ssr` 0.7.0 · `@supabase/supabase-js` 2.57.4 (both optional) |
| Validation | Zod 3.25 |
| UI | Framer Motion 14.3.0 · `lucide-react` 0.468.0 · Three.js 0.186.1 / `@react-three/fiber` 9.8.1 |
| Tests | Vitest 3.2.4 · `tsc --noEmit` typecheck · ESLint 9.39.0 |

No PyTorch, TensorFlow, LangChain, Pinecone, or vector database is used anywhere in this project. AI features are keyword retrieval, deterministic heuristics, and (teacher-side only) Gemini document understanding for question generation.

## Deployment

```mermaid
flowchart TB
    Learner([Learner browser]) --> Vercel["Vercel — Next.js 16 App Router<br/>devkraft-black.vercel.app"]
    Teacher([Teacher browser]) --> Vercel
    Vercel --> Pyodide["Pyodide Web Worker<br/>public/pyodide-worker.js · pinned<br/>Python runs in the browser"]
    Vercel --> Local["Browser localStorage<br/>arena event timeline + replay"]
    Vercel --> Supabase["Supabase Postgres + Auth<br/>11 tables · RLS enabled on all<br/>public key in browser · service-role key server-only"]
    Vercel --> Gemini["Gemini — server routes only<br/>teacher document understanding"]
    Vercel --> Ollama["Ollama — optional fallback<br/>OLLAMA_BASE_URL · OLLAMA_MODEL"]
    Vercel --> JDoodle["JDoodle / Judge0<br/>disabled by default · JDOODLE_EXECUTION_ENABLED=true opt-in"]
    Vercel -.-> DataDir[".data/ — local dev only<br/>ignored, not durable storage"]
```

- **Live demo** runs on Vercel at https://devkraft-black.vercel.app/ (Next.js 16 App Router).
- **Supabase is optional.** Without credentials the app runs locally with `.data/` files; with credentials it uses Postgres + Auth. The browser uses the Supabase public key; authoritative grading uses the server-only `SUPABASE_SERVICE_ROLE_KEY`, because student RLS must never expose answer keys.
- **Python runs in the browser** via the pinned Pyodide Web Worker (`public/pyodide-worker.js`). JDoodle / Judge0 are disabled by default and only opt in with `JDOODLE_EXECUTION_ENABLED=true` plus server-side credentials; arena execution through JDoodle has a 12-second request timeout.
- **Teacher document understanding** uses Gemini from server routes only (`GEMINI_API_KEY`), with Ollama (`OLLAMA_BASE_URL`, `OLLAMA_MODEL`) as an optional fallback when Gemini is unavailable or fails output/grounding validation. PDFs/PPTX still require Gemini for extraction.

## What works

- Responsive landing page with student, teacher, assessment, and coding arena navigation.
- Student dashboard with saved concept estimates and a deterministic next-mission recommendation when progress exists.
- Teacher assignment workspace with saved drafts, review, publishing, and honest empty analytics states.
- Server-served assessment questions and server-validated scoring with per-concept BKT mastery saved for an anonymous browser learner.
- Monaco code editor with TypeScript, JavaScript, and Python starter snippets. The arena can run visible samples and five hidden `countDown` cases through an explicitly enabled JDoodle provider; submissions update mastery.
- Arena attempts keep an event timeline in browser local storage and show a replay scrubber with integrity signals explicitly flagged for human review.
- Submitting a solution reveals an explanation viva generated from the submitted identifiers — even when the hidden tests fail or the execution provider is disabled; heuristic answers update mastery without sending source to an external API.
- Local concept-note keyword retrieval and separate deterministic mentor heuristics. No generated explanation is connected.
- Typed domain and API contracts, Supabase SSR/browser client helpers, and adaptive-learning unit tests.

## Start locally

```sh
npm install
cp .env.example .env.local
npm run dev
```

Then open `http://localhost:3000`. On Windows PowerShell, use `Copy-Item .env.example .env.local` instead of `cp`.

Local development can run without Supabase.

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
src/app/            Next.js pages and API routes
                    api/ · arena/ · assessment/ · learn/ · login/ · student/ · teacher/
src/components/     Landing, learning, coding, student, and teacher UI
src/features/       Adaptive learning (BKT, concept graph, missions) and assessments
src/lib/            Shared helpers (incl. Supabase SSR/browser clients)
src/config/         App configuration
src/data/           Concept notes and demo data
src/types/          Typed domain and API contracts
public/            favicon.png · js-worker.js · pyodide-worker.js
supabase/          SQL migrations for durable storage
docs/              Architecture, API contracts, setup, and team workflow
```

## Honest limits

- **No real authentication.** The anonymous learner cookie is a local-development identity, not an account system.
- **No sample data.** Every surface shows honest empty states until real learner or teacher activity is recorded.
- **No remote code execution on the live demo.** JDoodle and Judge0 are disabled unless explicitly configured with server-side credentials; Python runs locally in the browser via Pyodide.
- **No embeddings, vector database, or fine-tuning.** RAG is keyword retrieval; the mentor is deterministic heuristics.
- **No generated explanations connected by default.** The viva is generated from submitted identifiers with heuristic answers.
- **No persistent analytics yet.** Session replay, code viva, persistent analytics, and validated adaptive-model parameters remain future work.
- **Supabase is optional and local-first.** Without it, mastery and teacher state live in the ignored `.data/` directory, which is not durable storage.

## Team

Built by a team. Initial ownership areas, from `docs/TEAM_WORKFLOW.md`:

- Aman Shejul — frontend and integration
- Swapnil Thite — backend and execution
- Pankaj Dandekar — assessments, adaptive missions, and QA
- Yash Ambekar — AI mentor, RAG, and viva

This repository is an imported copy of the original team repository, [AmanShejul/devkraft](https://github.com/AmanShejul/devkraft). Commit history is preserved from the original.
