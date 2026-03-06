<div align="center">

<img src="./screenshots/logo.png" alt="WhisprBill Logo" width="180" />

# WhisprBill — Showcase Repository

### AI-powered GST-compliant invoicing platform for Indian businesses

[![Live App](https://img.shields.io/badge/Live%20App-app.whisprbill.com-0072E9?style=for-the-badge&logo=vercel)](https://app.whisprbill.com)
[![Landing](https://img.shields.io/badge/Landing-whisprbill.com-012652?style=for-the-badge&logo=next.js)](https://whisprbill.com)
[![Stack](https://img.shields.io/badge/Stack-MERN%20%2B%20Next.js-26363E?style=for-the-badge&logo=mongodb)](https://whisprbill.com)
[![Hosting](https://img.shields.io/badge/Backend-AWS%20Mumbai-FF9900?style=for-the-badge&logo=amazonaws)](https://whisprbill.com)
[![Frontend](https://img.shields.io/badge/Frontend-Vercel%20India-black?style=for-the-badge&logo=vercel)](https://whisprbill.com)

</div>

---

>  **This is a public showcase repository.** The production source code (frontend, app, backend) lives in private repositories. This repo contains architecture documentation, engineering decision writeups, and feature breakdowns intended for technical audiences — recruiters, collaborators, and senior engineers.

---

## What is WhisprBill?

WhisprBill is a production-grade, multi-tenant SaaS invoicing platform built specifically for Indian businesses. It handles the full invoicing lifecycle — from AI-assisted invoice creation to GST compliance, PDF generation, payment tracking, and business analytics — all in one place.

The product is live at [app.whisprbill.com](https://app.whisprbill.com).

**The core insight:** Most Indian SMBs struggle with GST compliance, manual data entry, and fragmented tools. WhisprBill collapses that into a single, intelligent workflow.

---

## Tech Stack

| Layer | Technology | Rationale |
|---|---|---|
| **App Frontend** | React, Zustand, Tailwind CSS | SPA for a snappy, stateful UX |
| **Landing Page** | Next.js (SSG), Tailwind CSS | SSG for maximum SEO performance |
| **Backend** | Node.js, Express.js | Familiar, fast, well-suited for REST APIs |
| **Database** | MongoDB | Flexible schema supports planned custom invoice fields; team expertise |
| **Auth** | Supabase (Google OAuth + email) | Managed auth, row-level security, India region |
| **AI Provider** | Groq API | Zero data retention policy, no model training on user data, and significantly lower per-invoice cost than alternatives |
| **Payments** | Razorpay | India-first payment gateway, UPI + cards |
| **PDF Engine** | Puppeteer + Handlebars | Dynamic HTML templating rendered to PDF server-side |
| **Frontend Hosting** | Vercel (India edge) | Native Next.js support, fast Indian PoP |
| **Backend Hosting** | AWS EC2 `ap-south-1` (Mumbai) | Low latency for Indian users, data residency in India |
| **CI/CD** | GitHub Actions + Docker + AWS ECR | Containerised deployments, clean image promotion |

---

## Architecture Overview

> 📁 See [`/architecture`](./architecture/) for all system diagrams.

```
                        ┌─────────────────────────────────────┐
                        │           CLIENT LAYER              │
                        │  whisprbill.com   app.whisprbill.com│
                        │  (Next.js SSG)    (React SPA)       │
                        └────────────┬────────────────┬───────┘
                                     │                │
                              Vercel India        Vercel India
                                     │                │
                        ┌────────────▼────────────────▼─────────┐
                        │         API LAYER (Express.js)        │
                        │         api.whisprbill.com            │
                        │         AWS EC2 ap-south-1            │
                        │                                       │
                        │  ┌──────────┐  ┌──────────────────┐   │
                        │  │ Auth MW  │  │ Rate Limiter (IP)│   │
                        │  └──────────┘  └──────────────────┘   │
                        │  ┌──────────────────────────────────┐ │
                        │  │ Route → Service → Domain Model   │ │
                        │  │ + Ownership checks per companyId │ │
                        │  └──────────────────────────────────┘ │
                        └──────┬──────────────────┬─────────────┘
                               │                  │
                    ┌──────────▼──────┐   ┌───────▼──────────┐
                    │   MongoDB       │   │  Groq API        │
                    │   (Primary DB)  │   │  (AI / LLM)      │
                    └─────────────────┘   └──────────────────┘
                               │
                    ┌──────────▼──────┐
                    │  Supabase       │
                    │  (Auth only)    │
                    └─────────────────┘
```

**All data is hosted within India.** The backend, database, and auth provider all operate in Indian regions, satisfying data residency requirements for Indian business customers.

---

## Engineering Highlights

These are specific technical decisions worth examining. Each has a dedicated writeup in [`/engineering-decisions`](./engineering-decisions/).

---

### 1. AI as a Parser, Never as Logic

**The problem:** LLMs are non-deterministic. In a financial application, you cannot let an AI decide tax rates, totals, or validation rules.

**The decision:** The AI layer is strictly an **intent parser and entity extractor**. It receives user input (text, voice, or structured fields), classifies intent, and returns structured JSON. From that point, the backend handles everything deterministically — GST calculations, totals, discount logic, validation, and DB writes.

**The architecture:**

```
User Input (text / voice / manual)
        │
        ▼
  ┌─────────────┐     ┌──────────────────┐     ┌──────────────────┐
  │promptBuilder│───▶ │  Groq API        │───▶│responseGenerator │
  └─────────────┘     │  (aiInterpreter) │     └────────┬─────────┘
                      └──────────────────┘              │
                                                Structured JSON
                                                        │
                                            ┌───────────▼────────────┐
                                            │  Backend Business Logic│
                                            │  Taxes · Totals ·      │
                                            │  Validation · DB Write │
                                            └────────────────────────┘
```

The orchestrator pattern (`promptBuilder`, `aiInterpreter`, `responseGenerator`) was a deliberate architectural choice to keep AI concerns fully isolated from financial logic. The AI can do full CRUD on invoices, customers, and inventory through natural language — but it never *executes* those operations itself.

---

### 2. PDF Blob Storage + Temporary URLs

**The problem:** Storing generated PDFs as physical files in object storage (e.g., S3) adds storage cost, requires lifecycle management, and creates a permanent link that could be misused.

**The decision:** PDFs are never stored as files. Instead, the Puppeteer-rendered PDF is stored as a **binary blob in MongoDB**. When a user needs to view or download an invoice, a **temporary URL** is generated on-demand, the blob is fetched, and the PDF is rendered in-browser. The URL expires after the session.

**Why this matters:**
- Zero file storage cost
- No permanent publicly accessible invoice URLs
- Simplifies deletion (one document delete removes all invoice data atomically)
- Works well at current scale; the tradeoff at higher scale (blob retrieval time) is acknowledged and the planned migration to S3 + signed URLs is already scoped

---

### 3. Automatic HSN Code + GST Slab Resolution

**The problem:** India's GST system has thousands of HSN codes across product families, with tax slabs (0%, 5%, 12%, 18%, 28%) varying by product type. Most invoicing tools make users look this up manually, which is a compliance risk.

**The decision:** When a user adds a product to inventory, WhisprBill **automatically resolves the HSN code and applicable GST slab** based on the product's family/category, sourced from official GoI regulations. The user never has to know which slab their product falls under.

This is one of the core compliance differentiators vs. generic invoicing tools.

---

### 4. Razorpay Webhook Idempotency + Transactional Handling

**The problem:** Payment webhooks from Razorpay can be delivered multiple times (at-least-once delivery). Without idempotency, a `payment.captured` event processed twice could double-credit a payment or corrupt invoice state.

**The decision:**
- Every incoming webhook is checked against a stored `paymentId` before processing
- The entire state update (invoice status + payment record) is wrapped in a **MongoDB transaction**
- Concurrent webhook delivery is handled with explicit concurrency guards
- IP-level rate limiting prevents webhook endpoint abuse

This ensures exactly-once semantics on payment state transitions even under retried or concurrent delivery.

---

### 5. Multi-Tenant Isolation via CompanyId Middleware

**The problem:** A single user can manage multiple companies on WhisprBill. Every API request must be scoped to the correct company and must never allow cross-company data access.

**The decision:** Every request carries a `companyId`. An ownership-check middleware validates that the requesting user is an authorised member of that company before any service layer code runs. This check happens at the route level, not buried inside business logic, so it cannot be accidentally bypassed.

---

### 6. Groq as AI Provider — A Cost-First Decision

This was a deliberate **product decision, not just a tech one**. The per-invoice AI cost at scale matters more than prestige of provider. Groq offered:
- Significantly lower inference cost vs. OpenAI/Anthropic
- **Zero data retention** and no training on customer data — critical for a financial SaaS
- Fast inference (important for a chat-like AI invoice flow)

The goal was never to build the most architecturally impressive system — it was to build something users can afford to use repeatedly. Reducing per-invoice AI cost directly impacts pricing strategy.

---

### 7. SSG Landing Page for SEO (Next.js on Vercel)

The marketing site (`whisprbill.com`) is fully separate from the app (`app.whisprbill.com`) and is built with **Next.js using Static Site Generation (SSG)**. Every page is pre-rendered at build time, giving optimal Core Web Vitals and crawlability.

This was a deliberate call to not bundle the landing and app — they have entirely different performance, SEO, and deployment requirements.

---

## Feature Breakdown

### 🧾 Invoicing
- Manual invoice creation with full field control
- AI-assisted invoice creation via text, voice, or hybrid input
- GST-compliant invoices (CGST / SGST / IGST) with automatic tax calculation
- Automatic HSN code + GST slab resolution by product family
- PDF generation with custom Handlebars templates
- Public invoice sharing via shareable link (no login required for recipient)

### 🤖 AI & Productivity
- Natural language CRUD on invoices, customers, and inventory
- Voice input support for invoice creation
- Intent classification (`aiInterpreter`) + natural language responses
- AI productivity dashboard tab (time saved, invoices generated via AI)

### 👥 Customer & Inventory Management
- Full customer management (create, update, bulk import via Excel)
- Inventory management with bulk Excel import/export
- Automatic HSN + GST slab assignment on inventory items

### 🏢 Multi-Company Support
- One user account can manage multiple business profiles / GSTINs
- Full data isolation between companies via companyId middleware

### 📊 Dashboard & Analytics
5-tab analytics dashboard covering:
- **Financials** — revenue, outstanding, paid invoices
- **Profitability** — margin trends
- **AI & Productivity** — time saved, AI usage metrics
- **Operations** — invoice volume, status breakdown
- *(+ more tabs in development)*

### 🔐 Security
- Google OAuth + email auth via Supabase
- Two-Factor Authentication (2FA)
- Webhook signature verification (Razorpay)
- IP-level rate limiting across all endpoints
- Idempotent payment processing
- All data hosted in India

### 💳 Payments
- Razorpay integration for subscription billing
- Transactional webhook handling with idempotency

---

## What We're Building Next

These are scoped and actively in development — not wishlist items:

| Improvement | Why |
|---|---|
| **Bull + Redis queue for PDF generation** | Prevent OOM on the backend instance under concurrent PDF load; proper job queue with retries |
| **Redis caching** | Reduce MongoDB reads on hot paths; reduce Groq token costs on repeated contexts |
| **PDF microservice** | Decouple Puppeteer from the main API server; independent scaling |
| **CDN / PoP layer** | India-first, then expand; improve static asset delivery speed |
| **AI fallback handling** | Graceful degradation when Groq returns malformed output |
| **Concurrent processing** | Extend transactional + async patterns to remaining high-volume flows |

---

## Repository Structure

```
whisprbill-showcase/
├── README.md
├── architecture/
│   ├── system-overview.png          # High-level system diagram
│   ├── ai-pipeline.png              # AI orchestrator flow
│   ├── pdf-generation-flow.png      # Current + planned queue architecture
│   └── webhook-flow.png             # Razorpay idempotency flow
├── engineering-decisions/
│   ├── pdf-blob-vs-s3.md
│   ├── ai-as-parser-not-logic.md
│   ├── webhook-idempotency.md
│   ├── mongodb-schema-design.md
│   └── ssg-landing-separation.md
└── screenshots/
    ├── landing.png
    ├── dashboard.png
    ├── invoice-creation.png
    └── ai-chat.png
```

## Live Links

| | |
|---|---|
| 🌐 Landing | [whisprbill.com](https://whisprbill.com) |
| 🚀 App | [app.whisprbill.com](https://app.whisprbill.com) |

---

<div align="center">
<sub>Source code is private. This showcase repo exists to document engineering decisions and architecture for technical audiences.</sub>
</div>
