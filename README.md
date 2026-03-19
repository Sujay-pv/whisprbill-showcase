<div align="center">

<img src="./screenshots/logo.png" alt="WhisprBill Logo" width="180" />

# WhisprBill - Showcase Repository

### AI-powered GST-compliant invoicing platform for Indian businesses

<p>
  <a href="https://whisprbill.com">Landing</a> •
  <a href="https://app.whisprbill.com">App</a> 
</p>

</div>

---

>  **This is a public showcase repository.** The production codebase is private.
  This repo focuses on architecture, engineering decisions, and system design for WhisprBill.

---

## What is WhisprBill?

WhisprBill is a multi-tenant SaaS invoicing platform built for Indian businesses.

It supports the full invoicing workflow, from AI-assisted invoice creation to GST-compliant document generation, payment tracking, and analytics.

The product is live at https://app.whisprbill.com.

The core problem it addresses is reducing manual effort and compliance friction for small and medium businesses handling GST-based invoicing.

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
| **Frontend Hosting** | Vercel and Netlify (India edge) | Native Next.js support, fast Indian PoP |
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
                              Vercel India        Netlify India
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

## CI/CD & Deployment Pipeline

The backend follows a fully containerised deployment pipeline with zero-touch deploys on every push to `main`.

```
Developer Push (main branch)
        │
        ▼
┌───────────────────┐
│  GitHub Actions   │  ← Triggered on push
│  Workflow         │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Docker Build     │  ← Builds production image
│  (Dockerfile)     │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  AWS ECR          │  ← Image pushed to Elastic Container Registry
│  (ap-south-1)     │     with commit SHA tag
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  AWS EC2          │  ← Instance pulls latest image from ECR
│  (Mumbai)         │     and restarts the container
└───────────────────┘
```

**How it works:**
- Every merge to `main` triggers a GitHub Actions workflow
- The workflow builds a Docker image and pushes it to **AWS ECR** tagged with the commit SHA
- The EC2 instance pulls the new image and performs a rolling container restart
- The frontend deploys automatically via **Vercel's Git integration** on push, no workflow needed

This gives the backend a clean, reproducible build environment and makes rollbacks trivial, just redeploy a previous ECR image tag.

---

## Engineering Highlights

These decisions reflect how the system is designed today, along with known tradeoffs and future improvements.

Key technical decisions in WhisprBill. Detailed writeups will be available in  [`/engineering-decisions`](./engineering-decisions/).

---

### 1. AI as a Parser, Not Business Logic

**Problem:** LLMs are non-deterministic. In a financial system, they cannot be trusted to handle tax calculations, totals, or validation.

**Decision:** The AI layer is used only for intent parsing and entity extraction. It converts user input into structured JSON. All financial logic - GST calculation, totals, validation, and persistence is handled by deterministic backend code.

**Flow:** 

```
User Input (text / voice / manual)
        │
        ▼
  ┌─────────────┐      ┌──────────────────┐     ┌──────────────────┐
  │promptBuilder│───▶ │  Groq API         |───▶│responseGenerator │
  └─────────────┘      │  (aiInterpreter) │     └────────┬─────────┘
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

### 6. AI Provider Selection (Groq)

This was primarily a product decision.

- Lower inference cost compared to alternatives
- Zero data retention and no training on user data
- Fast response times for chat-like workflows

The focus was to keep per-invoice AI cost low, making the product viable at scale.

---

### 7. SSG Landing Page (Next.js on Vercel)

The marketing site (`whisprbill.com`) is built with Next.js using Static Site Generation.

The app (`app.whisprbill.com`) is separate, since it has different performance and deployment requirements.

SSG improves SEO, load performance, and Core Web Vitals for the marketing layer.
---

## Feature Breakdown

###  Invoicing
- Manual invoice creation with full field control
- AI-assisted invoice creation via text, voice, or hybrid input
- GST-compliant invoices (CGST / SGST / IGST) with automatic tax calculation
- Automatic HSN code + GST slab resolution by product family
- PDF generation with custom Handlebars templates
- Public invoice sharing via shareable link (no login required for recipient)

###  AI & Productivity
- Natural language CRUD on invoices, customers, and inventory
- Voice input support for invoice creation
- Intent classification (`aiInterpreter`) + natural language responses
- AI productivity dashboard tab (time saved, invoices generated via AI)

###  Customer & Inventory Management
- Full customer management (create, update, bulk import via Excel)
- Inventory management with bulk Excel import/export
- Automatic HSN + GST slab assignment on inventory items

###  Multi-Company Support
- One user account can manage multiple business profiles / GSTINs
- Full data isolation between companies via companyId middleware

###  Dashboard & Analytics
5-tab analytics dashboard covering:
- **Financials** — revenue, outstanding, paid invoices
- **Profitability** — margin trends
- **AI & Productivity** — time saved, AI usage metrics
- **Operations** — invoice volume, status breakdown
- *(+ more tabs in development)*

###  Security
- Google OAuth + email auth via Supabase
- Two-Factor Authentication (2FA)
- Webhook signature verification (Razorpay)
- IP-level rate limiting across all endpoints
- Idempotent payment processing
- All data hosted in India

###  Payments
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

## Repository Structure (In progress)

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

- https://whisprbill.com
- https://app.whisprbill.com

---

<div align="center">
<sub>Source code is private. This showcase repo exists to document engineering decisions and architecture for technical audiences.</sub>
</div>
