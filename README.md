<div align="center">

<br/>

<img src="logo.png" alt="Tawasul Logo" width="120" />

<br/>

# تواصل · Tawasul

### Enterprise Multi-Tenant AI Sales Agent Platform

*From anonymous visitor to qualified lead to confirmed meeting — fully automated, deeply configurable, production-hardened.*

<br/>

[![Next.js](https://img.shields.io/badge/Next.js-App_Router-000000?style=flat-square&logo=next.js)](https://nextjs.org)
[![TypeScript](https://img.shields.io/badge/TypeScript-Strict-3178C6?style=flat-square&logo=typescript)](https://www.typescriptlang.org)
[![MongoDB](https://img.shields.io/badge/MongoDB-Mongoose-47A248?style=flat-square&logo=mongodb)](https://www.mongodb.com)
[![Groq](https://img.shields.io/badge/LLM-Groq_%2B_Gemini-F56565?style=flat-square)](https://groq.com)
[![Redis](https://img.shields.io/badge/Redis-Upstash-DC382D?style=flat-square&logo=redis)](https://upstash.com)
[![Vercel](https://img.shields.io/badge/Deployed-Vercel-000000?style=flat-square&logo=vercel)](https://vercel.com)
[![License](https://img.shields.io/badge/License-Proprietary-red?style=flat-square)](#)

<br/>

[**Live Demo**](https://aitawasol.com) · [Architecture](#architecture) · [API Reference](#api-reference) · [Deployment](#deployment)

<br/>

</div>

---

## What Is Tawasul?

Tawasul is a **production-grade, multi-tenant SaaS platform** that deploys fully-configured AI sales agents for businesses in any industry. Each agent is branded to the client company, trained on their specific offerings, and guided by a deterministic workflow engine that prevents hallucination and enforces business logic on every single turn.

Unlike off-the-shelf chatbots, Tawasul's agents don't just answer questions — they **qualify leads** against a 100-point multi-dimensional scoring model, **verify contact information** via email and WhatsApp OTP, **book meetings** directly into a slot-managed calendar, **generate AI-authored executive reports** ready for the sales team's review, and **automate post-qualification follow-up** with personalized multichannel sequences, pre-meeting intelligence briefs, and daily pipeline digests.

The platform ships with three independent control planes:

| Panel | Who Uses It | What It Does |
|-------|-------------|--------------|
| `/t/[slug]` | End visitors (prospects) | Conversational AI agent — the product itself |
| `/admin` | Tenant companies | CRM dashboard, lead pipeline, visitor analytics, agent configuration |
| `/super` | Platform operator | Multi-tenant management, platform-wide analytics, billing packages, industry sectors |

Both operator panels (`/admin` and `/super`) ship with a built-in **light / dark theme toggle**, remembered per operator for a flash-free render, with live trend sparklines, a real-time activity feed, and ready-to-send onboarding messages (with QR code) when provisioning a new company.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        VISITOR BROWSER                          │
│                 /t/acme-corp  (tenant landing page)             │
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTPS (rate-limited, CSP-protected)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              NEXT.JS APP ROUTER  (Node.js / Edge)               │
│                                                                 │
│  proxy.ts ──► Rate Limit (per-IP throttling on auth + AI)       │
│                                                                 │
│  POST /api/v1/public/{slug}/{agentKey}/sessions                 │
│  POST /api/v1/public/{slug}/{agentKey}/sessions/{id}/messages   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AI ORCHESTRATION ENGINE                      │
│                                                                 │
│  1. Tenant config load     → MongoDB (branding, agent config)   │
│  2. Workflow evaluation    → Next required field / action       │
│  3. Knowledge retrieval    → Semantic search on tenant docs     │
│  4. Prompt assembly        → Dynamic system prompt build        │
│  5. LLM inference          → Primary model with auto-failover   │
│  6. Structured extraction  → Schema-validated JSON output       │
│  7. Scoring evaluation     → 100-pt multi-dimension score       │
│  8. Field persistence      → Granular lead data storage         │
│  9. Webhook delivery       → HMAC-signed event to tenant CRM    │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                ┌───────────┴───────────┐
                ▼                       ▼
         MongoDB Atlas              Upstash Redis
         (30 models)           (rate limits, OTP cache,
                                token blacklist, slot locks)
```

### AI Turn Lifecycle

Every message sent by a visitor goes through an 9-step deterministic pipeline before a response is returned:

```
User Message
     │
     ├─► 1. Session Auth & State Load
     ├─► 2. Slash Command Detection  (built-in session control commands)
     ├─► 3. Parallel Lookups         (tenant, session, offerings — Promise.all)
     ├─► 4. Workflow Engine          (determine next target_field or action)
     ├─► 5. Knowledge Retrieval      (semantic search on tenant knowledge base)
     ├─► 6. Prompt Assembly          (persona, guardrails, conversation history, knowledge context)
     ├─► 7. LLM Inference            (primary model + automatic failover)
     ├─► 8. Structured Extraction    (Zod-validated JSON from LLM output)
     └─► 9. Persistence Pipeline     (messages, lead, field values, score snapshot)
          │
          └─► Webhook Dispatch (if lead.hot / meeting.booked event fires)
```

---

## Feature Breakdown

### Conversational AI Engine

- **Multi-State Conversation Machine** — each session transitions deterministically through a sequence of well-defined states covering greeting, qualification, clarification, summarization, booking, verification, and completion. State transitions are enforced by the workflow engine and cannot be skipped.
- **Workflow Engine** — directed graph with multiple node types and conditional rule operators. Includes infinite-loop detection and deterministic branching logic.
- **Dynamic Schema Generator** — builds Zod schemas on the fly from `FieldDefinition` records stored in MongoDB, supporting primitive types, enumerations, arrays, and domain-specific range types.
- **RAG Knowledge Layer** — semantic search across tenant's uploaded `KnowledgeSource` records (FAQ, product spec, policy, company info). Top results injected into system prompt with citation tracking.
- **Dual-Model Fallback** — Groq is the primary inference engine. The orchestrator automatically switches to a secondary model on transient errors, providing zero downtime for the end visitor.
- **Structured Extraction** — every LLM response is parsed for 20+ named fields (contact info, company profile, pain points, budget, timeline, decision makers) using strict Zod validation with empty-string-to-undefined preprocessing.
- **Slash Commands** — visitors can use built-in session control commands to manage their conversation state.

### Lead Scoring & Intelligence

- **100-Point Multi-Dimensional Scoring** — six independent scoring dimensions, each with configurable weight: *completeness*, *pain clarity*, *timeline urgency*, *budget signal*, *stakeholder seriousness*, *engagement quality*
- **Dynamic Scoring Policies** — scoring rules are stored as metadata in `ScoringPolicy` documents and evaluated at runtime. No code changes required to adjust scoring logic per tenant.
- **Score Buckets** — `sales_ready`, `nurture`, `disqualified`, `discovery_in_progress` — automatically assigned and updated on every turn.
- **LeadScoreSnapshot** — immutable historical record of every score evaluation, enabling full audit trail of a lead's qualification journey.
- **Granular Field Values** — `LeadFieldValue` stores each extracted data point with its confidence score (0–1), source (`ai_extraction` / `manual_entry` / `system_default`), and verification status.
- **Session Recency Signal** — the engagement quality dimension is boosted for leads whose conversation started recently, ensuring freshly-active prospects surface higher in the pipeline without any manual intervention.

### Booking & Calendar System

- **Slot Management** — tenant admins define available time slots with timezone support (Africa/Cairo default), duration, host name, and optional meeting URL.
- **Recurring Slots** — `is_recurring` flag with `recurrence_rule` for weekly availability patterns.
- **Idempotent Bookings** — `idempotency_key` prevents double-booking from network retries.
- **Booking State Machine** — `slot_selected → email_verification_pending → phone_verification_pending → confirmed → failed / cancelled / no_show`
- **Slot Locking** — Redis-backed distributed locks prevent race conditions when multiple prospects attempt to book the same slot simultaneously.
- **Meeting Reminders** — scheduled cron job dispatches confirmation emails and pre-meeting reminders to prospects automatically at the appropriate time windows.

### Contact Verification

- **Streamlined Contact Capture** *(default)* — visitors submit their name, phone, and email through a single friction-free form. On success, an animated confirmation screen is displayed immediately and the lead record is created in the tenant's pipeline without delay. This is the out-of-the-box experience for every company.
- **Email OTP** — short-lived time-limited code, attempt-capped, stored as a hashed value in Redis — never in plaintext.
- **WhatsApp OTP (premium)** — native WhatsApp message delivery via an integrated OTP provider, offered as a **package-gated capability**. The platform operator enables it per company only when their plan includes it; otherwise the streamlined direct-capture flow is used. The full verification flow is preserved end-to-end and activates automatically once turned on for an eligible tenant.
- **VerificationToken model** — tracks code hash, expiry, attempt count, and verification timestamp with MongoDB TTL auto-expire.

### AI Report Generation

- **6 Report Types** — `executive_summary`, `discovery_insight`, `qualification`, `recommendation`, `meeting_brief`, `followup_sequence`
- **Gemini 2.0 Flash powered** — structured JSON output with talking points, risk factors, deal size estimation, recommended next steps, and confidence scoring.
- **Bilingual Prompts** — separate Arabic and English prompt templates for each report type.
- **PDF Export** — `@react-pdf/renderer` generates print-ready PDF documents with lead data, score breakdown, and AI-authored insights.
- **Dependency Validation** — `dependency-validator.ts` ensures all required fields are populated before report generation begins, preventing incomplete output.
- **Token Cost Tracking** — `tokens_used` recorded per report for billing and cost optimization.

### Visitor Analytics & Tracking

- **First-Party Visitor Identity** — every visitor to any tenant page or platform panel receives a persistent anonymous identifier stored as a first-party cookie. No third-party scripts involved; entirely self-hosted.
- **Event Stream** — page views, chat sessions, form submissions, and booking attempts are captured as structured events with page path, referrer, device type, and timestamp. No PII is stored — IP addresses are one-way hashed.
- **Per-Tenant Analytics Dashboard** (`/admin/analytics`) — each tenant admin has an isolated view of their own audience: unique visitors (daily / 30-day), page views, top pages with a visual bar chart, daily visitor sparkline for the past 7 days, device split (mobile vs. desktop), and a live event feed with the last 30 events.
- **Platform-Wide Analytics** (`/super/analytics`) — the super admin sees aggregated metrics across all tenants and platform pages: total unique visitors, top visited pages, event breakdown by type, and a full event table with per-event tenant attribution.
- **Zero Visibility to End Users** — tracking is completely transparent to visitors. No consent banner is surfaced; no data is shared externally.
- **`VisitorEvent` Collection** — lightweight MongoDB collection with compound indexes on `(visitor_id, created_at)` and `(tenant_id, created_at)` for sub-millisecond query performance even at scale.

### Sales Intelligence & Automation

- **Follow-up Sequence Generation** — for any qualified lead, the platform generates a fully-structured 7-step multichannel outreach sequence spanning 30 days: Email (Day 1) → WhatsApp (Day 3) → Phone (Day 7) → Email (Day 10) → WhatsApp (Day 14) → Phone (Day 20) → Email (Day 30). Each email step includes a subject line and an A/B variant; each phone step includes a voicemail script. Sequences are personalized to the lead's profile, industry, pain points, and qualification score, rendered in Arabic or English based on the tenant's language setting, and stored as a `followup_sequence` LeadReport for the sales team to reference and act on.
- **Auto Pre-Meeting Brief** — 60 minutes before each confirmed meeting, the platform auto-generates an intelligence brief and delivers it to all `tenant_admin` and `sales_manager` users via email. The brief contains: lead background and company profile summary, pain point analysis, a recommended conversation opening line, 4–6 discovery questions tailored to the lead's situation, 2–3 anticipated objections with suggested responses, key talking points, deal potential assessment, and a clear meeting goal statement. The brief is simultaneously saved as a `LeadReport` (type `meeting_brief`) for permanent reference in the lead's timeline.
- **Daily EOD Digest** — every evening (6 PM Cairo time), each active tenant receives an automated day-end summary covering: new leads created that day, hot leads currently in the pipeline (score above threshold, not yet booked or disqualified), bookings confirmed today, and stale qualified leads that have not been updated in 3 or more days. Delivered to all `tenant_admin` and `sales_manager` users; automatically skipped for tenants with no activity that day. Fully bilingual based on the tenant's language configuration.

### Multi-Tenant Architecture

- **Unlimited tenants** — each with independent configuration: branding, color palette, theme mode, agent persona, guardrails, discovery flow, closing message, language (`ar` / `en`), regional dialect, and tone.
- **Per-Company Visual Identity** — every tenant defines a three-color brand palette (primary, secondary, accent), a default light or dark appearance, and whether visitors may switch between modes. The public agent page renders fully in the company's identity, anchored by a polished frosted-glass conversation panel that adapts to the chosen colors. The operator configures all of this at the moment a company is created.
- **Regional Dialect Tuning** — beyond base language, each agent can be set to a specific regional voice (Gulf, Saudi, Qatari, Emirati, Egyptian, or Levantine Arabic; British or American English) so the AI's phrasing matches the company's local audience.
- **Multi-Language Storefronts** — a company's public agent page can be published in several languages (12 supported, including Arabic, English, French, German, Turkish, Urdu, Persian, and more). Branding copy is translated with AI assistance and reviewed/approved by the operator before it goes live; visitors switch language from a built-in picker, and the page direction flips automatically between RTL and LTR. The visitor's choice is remembered server-side for a flash-free render.
- **Sector Classification** — tenants are organized by industry sectors (e.g., Real Estate, Healthcare, Manufacturing, Retail) for platform-wide analytics and reporting.
- **Package-Based Feature Gating** — `Package` model defines per-tier limits: `monthly_conversations`, `monthly_bookings`, `ai_report_credits`, `team_users`, `offerings_count`, and feature flags (`phone_verification`, `pdf_reports`, `advanced_analytics`, `white_label`, `api_access`, `voice_mode`). A company can only enable a capability — such as WhatsApp verification — that its package permits.
- **Industry Packs** — pre-built configuration bundles for specific verticals (ERP, Real Estate, etc.) that atomically install: FieldDefinitions, QuestionTemplates, WorkflowDefinition, ScoringPolicy, AgentInstance, and AgentVersion in a single MongoDB ACID transaction.
- **Immutable Agent Versioning** — `AgentVersion` bundles a `workflow_id` and `scoring_policy_id` into a locked, published snapshot. Production conversations are pinned to the version active at session creation. Admins can safely iterate without affecting live sessions.

### Webhook System

- **4 Event Types** — `lead.created`, `lead.qualified`, `lead.hot`, `meeting.booked`
- **HMAC-SHA256 Signatures** — every webhook payload is signed. Receiving servers can validate authenticity using the tenant's webhook secret.
- **Automatic Retry** — exponential backoff with progressive delays on delivery failure.
- **Per-Attempt Timeout** — hard timeout per delivery attempt prevents slow receivers from blocking the queue.
- **Per-Tenant Configuration** — webhook URL, secret, enabled events, and on/off toggle managed per tenant.

### API Key System

- **`twsl_` Prefixed Keys** — 32-byte cryptographically random keys with recognizable prefix.
- **SHA-256 Storage** — raw key is never stored; only the hash is persisted in MongoDB.
- **Truncated Display** — only the first 12 characters shown in UI after initial creation.
- **Granular Permissions** — `leads:read`, `leads:write`, `sessions:read`, `webhooks:send`
- **Expiry & Revocation** — optional `expires_at` date and hard `revoked` flag per key.
- **Last-Used Tracking** — fire-and-forget timestamp update on each successful verification.

### Security

| Layer | Implementation |
|-------|---------------|
| Authentication | JWT + httpOnly `strict` SameSite cookie |
| 2FA | TOTP via otplib — QR code setup with time-based epoch tolerance |
| Token Revocation | Redis blacklist on logout — JTI-based per-token invalidation |
| Password Hashing | bcryptjs — high-cost adaptive hashing |
| Rate Limiting | Multi-layer per-IP throttling — authentication endpoints and AI conversation API are independently protected |
| RBAC | 6-tier role system enforced at every API route |
| Input Validation | Zod on every API endpoint — no unvalidated data reaches business logic |
| CSP | Full Content-Security-Policy header — `default-src 'self'` with explicit allowlists |
| Security Headers | HSTS (2yr preload), X-Frame-Options DENY, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, X-XSS-Protection |
| API Keys | SHA-256 hashed, never stored raw, permission-scoped |
| Webhooks | HMAC-SHA256 payload signing |
| Audit Logging | Every critical action recorded with actor, entity, before/after state, IP address |
| Tenant Isolation | Tenant context validated at request level — cross-tenant data access impossible by design |

---

## Data Architecture

The platform is built on **30 Mongoose models** organized into 8 logical layers:

### Multi-Tenancy Core
| Model | Purpose |
|-------|---------|
| `Tenant` | Main organization entity — branding, agent config, webhook, usage counters |
| `TenantUser` | Team member with role assignment and optional TOTP |
| `Package` | SaaS billing tier with feature flags and usage limits |
| `Sector` | Industry vertical for tenant classification and analytics |

### Lead Management
| Model | Purpose |
|-------|---------|
| `ProspectLead` | Core lead entity — contact, company profile, qualification status, score |
| `LeadFieldValue` | Granular extracted field with confidence and verification status |
| `LeadScoreSnapshot` | Immutable scoring history for audit and trend analysis |
| `LeadReport` | AI-generated analysis document with PDF and metadata |

### Conversation Runtime
| Model | Purpose |
|-------|---------|
| `ConversationSession` | Session state machine — multi-stage lifecycle, lead context, confidence tracking |
| `ConversationMessage` | Individual turn — role, content, quick replies, tokens, latency |

### Configuration Layer (Metadata-Driven)
| Model | Purpose |
|-------|---------|
| `FieldDefinition` | Dynamic data point schema with type, validation, PII classification |
| `QuestionTemplate` | AI prompt templates per field with tone and input mode |
| `WorkflowDefinition` | Directed graph — nodes, edges, rules, branching logic |
| `ScoringPolicy` | Weighted dimensions with per-field point rules and bucket thresholds |
| `ReportTemplate` | Report section definitions with binding/LLM/computed modes |
| `KnowledgeSource` | RAG document store — FAQ, product specs, policy documents |

### Agent Management
| Model | Purpose |
|-------|---------|
| `AgentInstance` | Active agent deployment per tenant |
| `AgentVersion` | Immutable production bundle (workflow + scoring locked together) |

### Booking System
| Model | Purpose |
|-------|---------|
| `BookingSlot` | Available time slot with timezone, recurrence, and locking |
| `MeetingBooking` | Confirmed meeting — state machine, verification, reminders tracking |
| `VerificationToken` | OTP code with hash, TTL, attempt tracking, auto-expire |

### Catalog & Platform
| Model | Purpose |
|-------|---------|
| `Offering` | Product/service in the tenant's catalog — type, industries, budget range |
| `ApiKey` | External API credential — hashed, permission-scoped, expiring |
| `SiteConfig` | Singleton record for platform-level settings |

### Governance & Compliance
| Model | Purpose |
|-------|---------|
| `AuditLog` | Tamper-evident action log with before/after state and actor metadata |
| `NotificationLog` | Multi-channel delivery tracking — status, attempts, provider response |
| `FileUpload` | Uploaded document registry — size, type, storage key, origin |

### Analytics
| Model | Purpose |
|-------|---------|
| `VisitorEvent` | First-party visitor event stream — page views, interactions, device info, tenant attribution |

---

## API Reference

The platform exposes **55+ HTTP endpoints** across 9 namespaces. All protected namespaces require a valid session cookie or Bearer token and enforce role-based access at the handler level.

### Public Widget API  `/api/v1/public/`

The only unauthenticated namespace — exposed to end visitors through the chat widget.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/public/[slug]/[agentKey]/sessions` | Initialize a new AI conversation session |
| `POST` | `/api/v1/public/[slug]/[agentKey]/sessions/[id]/messages` | Send a message and receive an AI response |

Both endpoints are rate-limited per IP and return standard `X-RateLimit-*` headers.

### Authentication  `/api/auth/`  *(session-scoped)*

Full login/logout cycle with optional TOTP 2FA — credential verification, token issuance, Redis-based revocation on logout, and a complete 2FA setup/enable/disable/verify flow.

### Admin Dashboard  `/api/admin/`  *(tenant_admin, sales_manager, sales_user, auditor)*

21 endpoints covering the full tenant operator surface: lead pipeline management (list, detail, status, PDF export, CSV export, AI report generation, follow-up sequence generation, follow-up dispatch), catalog offerings, calendar slots and bookings, external API key management, webhook configuration, and agent branding settings.

### Tenant Management  `/api/tenants/`  *(platform_super_admin)*

Company (tenant) lifecycle: creation and update with full branding, color palette, theme, language, dialect, and agent setup; team user invitation and role management; package assignment; and atomic industry pack installation via a single transactional endpoint.

### Super Admin  `/api/super/`  *(platform_super_admin)*

Platform-wide controls: billing package management (create / edit / archive), industry sector management, global platform configuration, and cross-tenant data exports — both lead data and a full company roster — delivered as Excel-ready CSV.

### V1 External API  `/api/v1/`  *(API key auth)*

Versioned REST API for external CRM integration — lead reads and writes, compliance audit log access, and agent version publishing. Authenticated via `twsl_`-prefixed API keys verified by SHA-256 hash comparison.

### Visitor Tracking  `/api/track/`  *(public)*

Anonymous event ingestion endpoint. Accepts page view and interaction events from the browser with no authentication required. IP addresses are hashed server-side before storage; no raw addresses are persisted.

### Infrastructure

Health probe, public package listing, cron-secret-protected meeting reminder dispatch (timely prospect reminders + auto pre-meeting briefs to the sales team), and daily EOD digest dispatch.

---

## Tech Stack

### Core
| Technology | Role |
|-----------|------|
| Next.js (App Router) | Full-stack React framework with RSC support |
| React | UI library |
| TypeScript (strict) | Type-safe development |
| MongoDB + Mongoose | Primary database with schema validation |
| Upstash Redis | Rate limiting, session locks, OTP cache, token revocation |

### AI & Intelligence
| Technology | Role |
|-----------|------|
| Groq (LLaMA) | Primary inference — ultra-low latency |
| Google Gemini 2.0 Flash | Fallback inference + report generation |
| Vercel AI SDK | Provider-agnostic LLM client with streaming |
| otplib | TOTP generation and verification |

### Security
| Technology | Role |
|-----------|------|
| jose | JWT signing, verification, JTI management |
| bcryptjs | Password hashing |
| qrcode | TOTP QR code generation |
| HMAC-SHA256 | Webhook payload signing |
| CSP + Security Headers | XSS, clickjacking, MIME sniffing prevention |

### Frontend & UI
| Technology | Role |
|-----------|------|
| Tailwind CSS | Utility-first styling |
| Radix UI (25+ primitives) | Accessible, headless UI components |
| Shadcn UI | Pre-composed component library |
| Framer Motion | Animations and transitions |
| Recharts | Data visualization |
| React Hook Form + Zod | Form validation |

### Infrastructure
| Technology | Role |
|-----------|------|
| Vercel | Deployment, edge functions, analytics |
| @vercel/analytics | Real user monitoring |
| @vercel/speed-insights | Core Web Vitals tracking |
| Sentry | Error tracking and performance monitoring |
| Resend | Transactional email delivery |
| akedly.io | WhatsApp OTP delivery |
| @react-pdf/renderer | Server-side PDF generation |

---

## Project Structure

```
├── app/
│   ├── api/
│   │   ├── auth/               # Login, logout, 2FA (setup/enable/disable/verify)
│   │   ├── admin/              # Lead, booking, offering, API key, settings management
│   │   ├── tenants/            # Tenant CRUD, user management, pack installation
│   │   ├── super/              # Sectors, site config, platform-wide exports
│   │   ├── v1/                 # Versioned external API (API key auth)
│   │   ├── cron/               # Scheduled jobs (meeting reminders, pre-meeting briefs, EOD digest)
│   │   ├── track/              # Anonymous visitor event ingestion
│   │   └── health/             # Readiness probe
│   ├── admin/                  # Tenant admin panel (leads, calendar, analytics, settings)
│   ├── super/                  # Platform super admin (tenants, sectors, packages, analytics)
│   ├── t/[tenantSlug]/         # Public tenant agent page
│   │   └── opengraph-image.tsx # Dynamic branded OG image (1200×630)
│   ├── login/                  # Auth page with TOTP step
│   ├── sitemap.ts              # Dynamic sitemap (fetches active tenant slugs)
│   └── layout.tsx              # Root layout (fonts, JSON-LD, analytics)
│
├── components/
│   ├── chat/                   # Frosted-glass chat widget, bubbles, contact modal, slot picker
│   ├── admin/                  # Sidebar, dashboard charts, score badge
│   ├── super/                  # Create/edit tenant modals, package manager, platform charts
│   ├── tenant/                 # Per-company theme wrapper (palette, light/dark, visitor toggle)
│   ├── landing/                # Hero, features, pricing, FAQ sections
│   ├── tracking/               # First-party visitor tracker (client component)
│   ├── charts/                 # Area, funnel, donut, activity charts
│   └── ui/                     # 40+ Radix/Shadcn primitives
│
├── lib/
│   ├── ai/
│   │   ├── orchestrator.ts     # Core LLM turn engine with dual-model fallback
│   │   ├── workflow-engine.ts  # Directed graph evaluator with loop detection
│   │   ├── knowledge-retriever.ts  # RAG document search and context injection
│   │   ├── schema-generator.ts # Dynamic Zod schema from FieldDefinition
│   │   ├── extractor.ts        # 20+ field structured extraction schema
│   │   ├── sequence-generator.ts # 7-step multichannel follow-up sequence generator
│   │   └── prompts.ts          # System prompt builders (tenant-aware)
│   ├── scoring/
│   │   └── engine.ts           # 100-pt multi-dimensional lead scoring
│   ├── packs/
│   │   ├── installer.ts        # Atomic ACID pack installation
│   │   ├── erp.ts              # ERP industry pack template
│   │   └── real-estate.ts      # Real Estate industry pack template
│   ├── auth/
│   │   ├── jwt.ts              # Token signing, verification, role-based type definitions
│   │   ├── totp.ts             # TOTP secret, QR, verification
│   │   ├── api-key.ts          # SHA-256 key hashing and verification
│   │   └── guards.ts           # Route-level auth and access control helpers
│   ├── redis/client.ts         # Upstash REST client with connection resilience
│   ├── webhooks/deliver.ts     # HMAC-signed webhook delivery with retry logic
│   ├── reports/generator.ts    # Gemini-powered AI report generation
│   ├── notifications/          # Email (Resend) + WhatsApp (akedly) delivery
│   ├── email.ts                # Lead notification templates
│   └── errors/index.ts         # Typed error hierarchy (15 error types)
│
├── models/                     # 30 Mongoose schemas (see Data Architecture above)
├── proxy.ts                   # Route protection + rate limiting (auth + public AI endpoints)
├── next.config.ts              # Security headers (CSP, HSTS, XFO, etc.)
├── public/
│   ├── robots.txt              # Blocks /admin, /super, /api from indexing
│   └── sitemap.xml             # Auto-generated by Next.js sitemap.ts
└── tsconfig.json               # Strict TypeScript (ES2020 target)
```

---

## Deployment

### Prerequisites

- Node.js 20+
- MongoDB Atlas cluster (dedicated cluster recommended for production)
- Upstash Redis account (free tier sufficient for development)
- Vercel account (or any Node.js hosting)
- Groq API key
- Google Gemini API key (for reports and fallback)

### Environment Variables

```bash
# Database
MONGODB_URI=<mongodb-connection-string>

# Auth
JWT_SECRET=<minimum-32-char-random-string>

# AI
GROQ_API_KEY=<your-groq-api-key>
GOOGLE_GENERATIVE_AI_API_KEY=<your-gemini-api-key>

# Redis (Upstash)
UPSTASH_REDIS_REST_URL=<your-upstash-redis-url>
UPSTASH_REDIS_REST_TOKEN=<your-upstash-redis-token>

# Email
RESEND_API_KEY=<your-resend-api-key>
EMAIL_FROM=<sender-name> <<notifications@yourdomain.com>>

# WhatsApp OTP delivery (used when a tenant's package enables phone verification)
AKEDLY_API_KEY=<your-akedly-api-key>
AKEDLY_PIPELINE_ID=<your-pipeline-id>
AKEDLY_WEBHOOK_SECRET=<your-webhook-secret>

# Monitoring
SENTRY_DSN=<your-sentry-dsn>
NEXT_PUBLIC_SENTRY_DSN=<your-sentry-dsn>
SENTRY_ORG=<your-sentry-org>
SENTRY_PROJECT=<your-sentry-project>
SENTRY_AUTH_TOKEN=<your-sentry-auth-token>

# Platform
CRON_SECRET=<strong-random-secret>
NEXT_PUBLIC_APP_URL=https://yourdomain.com
NODE_ENV=production
```

### Initial Setup

```bash
# 1. Install dependencies
npm install

# 2. Initialize the platform (creates the super admin account)
npx tsx scripts/seed-superadmin.ts

# 3. Seed demo data (optional)
npx tsx scripts/seed-phase2.ts

# 4. Run locally
npm run dev
```

### Production Deploy

```bash
# Deploy to Vercel
vercel --prod

# Or build manually
npm run build
npm start
```


---

## Key Design Decisions

**Why MongoDB over PostgreSQL?**  
The platform's configuration layer is deeply hierarchical and schema-flexible. Workflow nodes, scoring dimensions, field definitions, and report section configurations vary significantly per industry pack. MongoDB's document model handles this variability without complex JOIN chains, while Mongoose provides schema validation at the application layer.

**Why a deterministic workflow engine over pure LLM autonomy?**  
Hallucination in B2B sales contexts carries real business risk — an AI that invents product features or pricing destroys trust. The workflow engine acts as a guardrail: it tells the LLM exactly what information to collect next. The LLM handles natural language understanding and generation; the engine handles business logic. The combination gives the best of both worlds.

**Why immutable agent versions?**  
A live conversation session that's mid-qualification should not be affected by a workflow change the admin is testing. By pinning sessions to an `AgentVersion` at creation time, admins can safely iterate on configuration in parallel without disrupting ongoing conversations.

**Why a two-layer rate limiting architecture?**  
The platform enforces rate limiting at two independent layers to ensure protection scales correctly across serverless deployments. The first layer operates at the edge request level; the second provides distributed enforcement across all function instances via Redis, ensuring no instance can be singled out to bypass per-instance limits.

---

## Platform Screenshots

| View | Description |
|------|-------------|
| `/t/acme` | Dynamic tenant landing page — full brand palette, light/dark with visitor toggle, frosted-glass AI panel |
| `/admin` | Lead pipeline with score visualization and activity feed |
| `/admin/leads/[id]` | Lead drill-down with score breakdown, timeline, and field values |
| `/admin/analytics` | Tenant-scoped visitor analytics — unique visitors, top pages, device split, daily chart |
| `/admin/settings` | Agent configuration — persona, scope, discovery flow, webhooks, 2FA |
| `/super` | Platform dashboard with tenant count, sector breakdown, lead analytics |
| `/super/tenants` | Company management with search, status filter, package/sector overview, and Excel export |
| `/super/packages` | Billing package manager — create, edit, and archive plans with limits and feature flags |
| `/super/analytics` | Platform-wide visitor analytics with cross-tenant event attribution |
| `/super/sectors` | Industry sector CRUD with emoji icons |

---

## License

Proprietary software. All rights reserved.

---

<div align="center">

**Tawasul · تواصل**

Built with precision. Deployed at scale. Ready for any industry.

*[aitawasol.com](https://aitawasol.com)*

</div>
