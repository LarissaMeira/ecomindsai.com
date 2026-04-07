# Ecominds.AI — Technical Architecture

## 1. Platform Overview

### Problem & Audience

Impact-driven organizations (NGOs, social enterprises, ESG startups) struggle to assess project readiness, define measurable indicators, and find aligned funding opportunities. These tasks require specialized knowledge, are time-consuming, and typically depend on expensive consultants.

### Core Value Proposition

Ecominds.AI is an AI-powered co-pilot that guides impact project leaders through a structured diagnostic-to-funding pipeline. Users submit project details through a guided form, receive an AI-generated diagnostic analysis (problem framing, stakeholder mapping, SDG alignment, territorial analysis), auto-generated KPI indicators with tracking dashboards, and — in the upcoming phase — matched funding opportunities. The platform turns weeks of consultant work into minutes of autonomous AI processing, while keeping humans in control of editing and refining every output.

---

## 2. System Architecture

### High-Level Diagram

```
┌──────────────────────────────────────────────────────────┐
│                      CLIENT (SPA)                        │
│  React 18 + Vite + Tailwind + shadcn/ui + Framer Motion  │
│                                                          │
│  Pages: Landing → Auth → Dashboard → Diagnostico →       │
│         ResultadoDiagnostico → Indicadores →              │
│         DashboardMetricas → Projetos → StatusPage         │
└──────────────┬───────────────────────┬───────────────────┘
               │ Supabase JS SDK       │ supabase.functions
               │ (Auth, DB, Storage)   │ .invoke()
               ▼                       ▼
┌──────────────────────┐  ┌────────────────────────────────┐
│    SUPABASE CLOUD    │  │    SUPABASE EDGE FUNCTIONS     │
│                      │  │                                │
│  • PostgreSQL (RLS)  │  │  call-diagnostico-webhook      │
│  • Auth (email/pwd)  │  │  call-indicadores-webhook      │
│  • Storage (files)   │  │  (future: call-oportunidades)  │
│  • Realtime          │  │                                │
│  • DB Functions      │  │  Auth verification + ownership │
│  • Triggers          │  │  check before proxying to N8N  │
└──────────────────────┘  └──────────────┬─────────────────┘
                                         │ HTTPS POST
                                         ▼
                          ┌──────────────────────────────┐
                          │      N8N (Railway)           │
                          │                              │
                          │  Workflow: /diagnostico      │
                          │  Workflow: /indicadores      │
                          │  Workflow: /oportunidades    │
                          │                              │
                          │  Orchestrates LLM calls,     │
                          │  prompt chains, and response  │
                          │  formatting                  │
                          └──────────────────────────────┘
```

### Main Modules

| Module | Responsibility |
|--------|---------------|
| **Landing & Auth** | Public page, email/password authentication via Supabase Auth |
| **Dashboard** | Conditional rendering (empty state onboarding vs. active project list with progress tracking) |
| **Diagnóstico** | Multi-step guided form (4 sections), input validation (Zod), draft auto-save (sessionStorage), submission to Edge Function |
| **Resultado Diagnóstico** | Renders AI-generated analysis in editable blocks (problem, audience, SDG alignment, territorial analysis, strengths, challenges, recommendations). Users can edit and persist changes |
| **Indicadores** | Displays AI-generated KPIs, allows manual metric tracking with period-based data entry, inline editing |
| **Dashboard Métricas** | Recharts-based visualization layer over indicator metrics — configurable chart types, color palettes, combined charts, presentation mode, PDF/image export |
| **Projetos** | Project management hub — create status pages, share with collaborators, manage visibility |
| **Compartilhados Comigo** | Inbound shared projects — accept/reject invitations, view shared content |
| **Status Page** | Public-facing project status with configurable sections (roadmap, budget, timeline, Kanban), viewer access control |
| **Edge Functions** | Authenticated proxies that verify JWT + ownership before forwarding to N8N webhooks |

### Data Flow

```
User fills form → Client validates (Zod) → Supabase insert (diagnostics table, status: "pending")
  → Client calls Edge Function → Edge Function verifies auth + ownership
  → Edge Function calls N8N webhook → N8N orchestrates LLM → returns structured output
  → Edge Function normalizes response → Client parses and updates Supabase
  → Client renders result sections → User can edit inline → changes saved to DB
```

---

## 3. AI & Agentic Design

### LLM Orchestration

LLM calls are **not** made from the client or from Supabase directly. All AI orchestration happens in **N8N workflows** hosted on Railway. This provides:

- **Separation of concerns**: The React app knows nothing about prompts, models, or tokens
- **Workflow flexibility**: N8N allows visual editing of prompt chains, conditional branches, and retry logic without code deploys
- **Model agnosticism**: The LLM provider can be swapped in N8N without touching application code

### Pipeline Architecture

Each major feature maps to one N8N workflow:

1. **`/diagnostico`** — Receives structured form data → generates a comprehensive diagnostic analysis covering: problem/solution framing, target audience analysis, SDG alignment scoring, territorial context, strengths, challenges, and strategic recommendations
2. **`/indicadores`** — Receives diagnostic context (problem, audience, SDGs, strengths) → generates a table of KPI indicators with suggested targets, units, and measurement methods
3. **`/oportunidades`** (planned) — Will receive diagnostic + indicator context → match against funding opportunities database

### Prompt Architecture Decisions

- **System prompts**: Defined within N8N nodes — the application layer never constructs or modifies prompts
- **Context passing**: Each downstream workflow receives curated context from previous stages (not raw form data), enabling chain-of-thought reasoning across the pipeline
- **Output structure**: Webhooks return structured text sections delimited by known markers (e.g., `## Problema`, `## Público-Alvo`), which the client parses via regex extraction (`extractAndValidateSection`)
- **Validation**: Client-side Zod schemas (`webhookResponseSchema`) validate response structure before rendering — malformed outputs are caught and surfaced as errors

### Agent Output Consistency

- **Edge Function normalization**: Handles malformed N8N responses (e.g., double-nested JSON keys) before they reach the client
- **Section extraction with fallbacks**: Each diagnostic section is independently parsed — one failed section doesn't break the others
- **Max length constraints**: Zod schemas enforce `max(50000)` on output strings to prevent rendering issues
- **Editable outputs**: Users can correct any AI-generated text, making the system self-healing through human review

---

## 4. Automation Workflows

### Fully Autonomous Flows

| Flow | Trigger | What Happens |
|------|---------|-------------|
| **Diagnostic generation** | User submits form | Insert DB record → call Edge Function → N8N processes → parse result → update 8 DB columns → render |
| **Indicator generation** | User clicks "Generate" on diagnostic result | Edge Function verifies credits → N8N processes → parse KPI table → insert into `indicators` table |
| **Credit management** | Each generation step | `decrement_credits` DB function atomically decrements available credits |
| **Share → Viewer sync** | Share is accepted | `sync_share_to_viewers` trigger automatically grants status page access |
| **Comment notifications** | New comment on status page | `create_comment_notification` trigger creates notification for page owner and parent comment author |
| **Profile + credits init** | New user signup | `handle_new_user` trigger creates profile + initializes 3 free credits |

### Human-in-the-Loop Checkpoints

| Checkpoint | Why |
|-----------|-----|
| **Diagnostic form review** | User reviews 4-step form before submitting — ensures AI receives quality input |
| **Result editing** | Every AI-generated section is editable — users refine analysis with domain expertise |
| **Indicator data entry** | Users manually input metric values per period — AI suggests KPIs but humans track reality |
| **Share acceptance** | Invited collaborators must explicitly accept project invitations |
| **Status page configuration** | Project owners choose which sections are visible (diagnostic, indicators, dashboard, budget, roadmap) |

### Tool Roles

| Tool | Role |
|------|------|
| **React + Vite** | Single-page application with client-side routing, form management, real-time UI updates |
| **Supabase** | Auth (email/password), PostgreSQL with RLS, Edge Functions (auth proxy), Storage (attachments), Realtime (collaboration), DB Functions & Triggers (automation) |
| **N8N (Railway)** | LLM orchestration engine — receives structured input via webhooks, runs prompt chains, returns formatted analysis. Visual workflow editor enables non-developer iteration on prompts |
| **Lovable** | Development platform — AI-assisted code generation, preview, deployment, Supabase integration management |

---

## 5. Key Technical Decisions

### Chosen

| Decision | Rationale |
|----------|-----------|
| **Edge Functions as auth proxy** | N8N webhook URLs are never exposed to the client. Edge Functions verify JWT, check resource ownership, and validate credits before forwarding — defense in depth |
| **N8N for LLM orchestration** | Visual workflow editor allows rapid prompt iteration without code deploys. Supports branching, retries, and multi-step chains natively |
| **Client-side section parsing** | AI output arrives as structured markdown. Client extracts sections via regex into independent DB columns. This allows per-section editing, selective display, and partial failure tolerance |
| **RLS + Security Definer functions** | `has_role`-style functions (`is_status_page_owner`, `can_viewer_access`) bypass RLS recursion while enforcing authorization at the database level |
| **Credits system** | Atomic `decrement_credits` function prevents race conditions. Freemium model with 3 initial credits gates expensive LLM calls |
| **SessionStorage for draft persistence** | Form data survives page refreshes within a session but is cleared on tab close — balances UX with privacy |
| **Separate indicator_metrics table** | Time-series data (values per period per indicator) is stored independently from the AI-generated indicator definitions, enabling flexible tracking without touching AI outputs |

### Discarded / Iterated

| What | Why it changed |
|------|---------------|
| **Direct client→webhook calls** | Initially, the React app called N8N webhooks directly. Replaced with Edge Function proxies for security (auth, ownership, URL hiding) |
| **Single route for shared projects** | Had both `/compartilhado/:token` (old) and `/s/:token` (new). Unified to `/s/:token` — old route removed after causing navigation bugs |
| **Sidebar "Journey" navigation** | Originally had Diagnóstico, Indicadores, Dashboard, Oportunidades as sidebar links. Removed — these are now accessed contextually through project cards, reducing cognitive load |
| **Public share links** | Removed public link generation from share dialogs. Sharing is invite-only (by email), improving access control |
| **Oportunidades as static feature** | Initially planned as a simple display. Redesigned as a full N8N workflow pipeline that will consume diagnostic context — currently marked "Em breve" |

---

## 6. Stack Summary

| Layer | Technology |
|-------|-----------|
| **Frontend** | React 18, TypeScript 5, Vite 5, Tailwind CSS 3, shadcn/ui, Framer Motion, Recharts, React Router, TanStack Query |
| **Backend** | Supabase (PostgreSQL 15 + RLS, Auth, Edge Functions in Deno, Storage, Realtime) |
| **AI Layer** | N8N workflows on Railway — orchestrates LLM calls, prompt chains, and structured output formatting |
| **Automation** | PostgreSQL triggers (user init, share sync, notifications), DB functions (credit management, access control), Edge Functions (auth proxy) |
| **Validation** | Zod schemas for form input and webhook response validation |
| **Deployment** | Lovable (frontend hosting + CI), Supabase Cloud (backend), Railway (N8N) |

---

**Project URL**: [https://lovable.dev/projects/46849eaa-5626-4aab-88f7-989f7d820411](https://lovable.dev/projects/46849eaa-5626-4aab-88f7-989f7d820411)
**Live App**: [https://ecomindsai.lovable.app](https://ecomindsai.lovable.app)
