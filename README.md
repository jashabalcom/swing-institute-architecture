# Swing Institute — Architecture Reference

**Public architecture + decision record for a production baseball-training SaaS (web + iOS).**

The application code lives in a private repository. This repo exists so engineers, architects, and hiring managers can read the system design, the decisions behind it, and the build history without needing source access.

[![Live](https://img.shields.io/badge/live-swinginstitutebaseball.com-1B2A4A)](https://www.swinginstitutebaseball.com)
[![Stack](https://img.shields.io/badge/stack-AWS%20Bedrock%20%7C%20Supabase%20%7C%20Stripe%20Connect%20%7C%20Capacitor-FF9900)](#tech-stack)

---

## What this platform is

Swing Institute is a full-stack athlete-development platform used by MLB players, collegiate athletes, and youth prospects. It combines:

- An **AI swing-analysis engine** (phone video → AWS Bedrock Claude Sonnet 4 Vision → MLB-benchmarked biomechanical scorecard in ~8 s).
- A **coaching marketplace** with Stripe Connect Express payouts, GPS-verified check-in, and mutual session confirmation.
- An **on-demand academy** (Levels → Modules → Lessons, progress tracking).
- A **live-session WebRTC room** (Daily.co) with transcription.
- A **community layer** (posts, DMs, group chats, polls, moderation).
- A **parent dashboard** with COPPA-compliant child accounts.
- Four **partner-program landing pages** (Coach / NIL / Ambassador / Affiliate) with variant-driven components.
- A **20-page admin surface** including a launch-readiness dashboard with 13 automated checks.

Single TypeScript codebase → web (AWS Amplify + CloudFront) + iOS App Store (Capacitor 8) + PWA.

---

## Architecture at a glance

```mermaid
flowchart TB
    subgraph CLIENTS["Clients"]
        direction LR
        ios["iOS App<br/>(Capacitor 8)"]
        web["Web Browser<br/>(SPA)"]
    end

    subgraph EDGE["AWS Edge / CDN"]
        direction LR
        r53["Route 53<br/>DNS"]
        acm["ACM<br/>(TLS)"]
        cf["CloudFront<br/>(CDN)"]
        amp["Amplify Hosting"]
    end

    subgraph SUPABASE["Supabase Control Plane"]
        direction LR
        sbauth["Auth (JWT)"]
        sbfn["35 Deno Edge Functions"]
        sbstor["Storage Buckets"]
        sbrt["Realtime"]
    end

    subgraph DATA["Data Plane"]
        direction LR
        pg[("Postgres 15<br/>RLS, 89 migrations")]
        rpc[/"Atomic RPCs"/]
    end

    subgraph AIPLANE["AWS AI / Messaging"]
        direction LR
        bedrock["Bedrock<br/>Claude Sonnet 4 Vision"]
        ses["SES<br/>(SIG V4)"]
        apns["APNs HTTP/2"]
    end

    subgraph MONEY["Payments"]
        stripe["Stripe + Stripe Connect"]
    end

    ios --> cf
    web --> cf
    r53 --> cf
    acm --> cf
    cf --> amp
    amp --> sbauth
    sbauth --> sbfn
    sbfn --> pg
    sbfn --> sbstor
    pg -.-> rpc
    sbfn --> bedrock
    sbfn --> ses
    sbfn --> apns
    sbfn --> stripe
    pg -.-> sbrt
    sbrt -.-> web
    sbrt -.-> ios

    classDef awsClass fill:#FF9900,stroke:#232F3E,color:#000,stroke-width:1.5px
    classDef sbClass fill:#3ECF8E,stroke:#1F8F5C,color:#000,stroke-width:1.5px
    classDef stripeClass fill:#635BFF,stroke:#3D2BFF,color:#fff,stroke-width:1.5px
    classDef appleClass fill:#1d1d1f,stroke:#000,color:#fff,stroke-width:1.5px
    classDef pgClass fill:#336791,stroke:#1F4060,color:#fff,stroke-width:1.5px

    class r53,acm,cf,amp,bedrock,ses awsClass
    class sbauth,sbfn,sbstor,sbrt sbClass
    class stripe stripeClass
    class apns,ios appleClass
    class pg,rpc pgClass
```

Branded fill colors: AWS orange, Supabase green, Stripe purple, Postgres blue, Apple charcoal. Full architecture with 6 diagrams (logical, physical, AI sequence, payout sequence, notification fan-out, ERD) in [ARCHITECTURE.md](ARCHITECTURE.md).

> For a hero-grade static asset using the **official AWS Architecture Icons** (PNG/SVG from <https://aws.amazon.com/architecture/icons/>), build it once in [Excalidraw](https://excalidraw.com) or [draw.io](https://app.diagrams.net) (full AWS stencil pack), export, and embed inline.

---

## Read this repo in order

1. **[ARCHITECTURE.md](ARCHITECTURE.md)** — system design deep dive: logical architecture, physical AWS topology, request flows, data model, trust boundaries, scalability posture, observability.
2. **[ENGINEERING-DECISIONS.md](ENGINEERING-DECISIONS.md)** — 14 ADRs. Each decision has context, alternatives considered, trade-offs accepted, and current status.
3. **[BUILD-TIMELINE.md](BUILD-TIMELINE.md)** — 18-stage narrative of what shipped and why, drawn from 564 commits on `main`.
4. **[AWS-DEPLOYMENT-PLAN.md](AWS-DEPLOYMENT-PLAN.md)** — phased migration plan from the current Supabase/Amplify setup to AWS-native (Aurora Serverless v2, Lambda, API Gateway, Cognito) when scale demands it.

---

## Tech stack

| Concern | Choice |
|---|---|
| Frontend | Vite 6, React 18, TypeScript, TailwindCSS 3, shadcn/ui, TanStack Query, React Router v6 |
| Mobile | Capacitor 8 + native Swift plugins (Apple Vision pose detection, Sign in with Apple) |
| Backend | 35 Deno Edge Functions on Supabase |
| Database | Postgres 15 with Row-Level Security, 89 forward-only migrations, pg_cron for scheduled jobs |
| AI / ML | AWS Bedrock (Claude Sonnet 4 vision) via `aws4fetch` SIG V4; MediaPipe Tasks Vision (web); Apple `VNDetectHumanBodyPoseRequest` (iOS) |
| Payments | Stripe Checkout, Stripe Connect Express, Customer Portal, atomic credit RPCs, signed webhooks, daily reconciliation |
| Email | AWS SES with SIG V4 signing, 80+ branded transactional templates |
| Push | Apple APNs HTTP/2 direct, dual sandbox + production endpoint retry, auto dead-token cleanup |
| Live video | Daily.co WebRTC rooms + transcription |
| CRM | GoHighLevel sync on signup |
| Hosting | AWS Amplify + CloudFront + ACM + Route 53 |
| Observability | Sentry; custom launch-readiness dashboard with 13 automated checks |
| Testing | Vitest, Playwright (E2E on partner-application flows) |

---

## Scale and cost posture

**Launch tier (0–500 MAU):** ~$75–130/mo total across AWS + Supabase + Stripe (variable).

**Phase-2 target (500–5000 MAU):** migrate to AWS-native per [AWS-DEPLOYMENT-PLAN.md](AWS-DEPLOYMENT-PLAN.md). Order: frontend → auth → data → functions. AWS DMS handles logical replication during DB cutover.

---

## What this demonstrates

This codebase is the reference point for:

- **Solutions architecture** — multi-service AWS design, edge-first backend, phased cloud migration plan.
- **Production engineering** — atomic concurrency, webhook-as-source-of-truth payments, idempotent scheduled jobs, self-healing push delivery.
- **Product engineering** — one codebase to three platforms, hybrid on-device + cloud AI pipeline, RBAC with RLS enforced at the database layer.
- **Operational rigor** — 89 migrations without breakage, 14 ADRs, a launch-readiness dashboard that turns go/no-go into a single pane of glass.

---

## Author

**Jasha Balcom** — Solutions Architect & Full-Stack Engineer. Former Chicago Cubs prospect and MLB performance coach. AWS-certified cloud practitioner.

- Live product: [swinginstitutebaseball.com](https://www.swinginstitutebaseball.com)
- Private application repo: `jashabalcom/swinginsitute` (access on request)
