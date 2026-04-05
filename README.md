# Swing Institute — Production Architecture

> **AI-Enhanced Baseball Training SaaS**
> Live at [swinginstitute.com](https://www.swinginstitutebaseball.com) | Serving competitive athletes in Atlanta and nationwide

Production multi-tenant SaaS platform for baseball player development — video coaching, structured curriculum, real-time community, subscription billing, and event management. Built serverless-first on AWS.

---

## System Architecture

```
                           Internet
                              |
                    +-------------------+
                    |    Route 53       |
                    |    DNS + TLS      |
                    +--------+----------+
                             |
                    +--------v----------+
                    |   AWS Amplify     |
                    |   CloudFront CDN  |
                    |   React SPA       |
                    +--------+----------+
                             |
              +--------------+--------------+
              |                             |
    +---------v----------+       +----------v-----------+
    |   Auth Layer       |       |   9 Edge Functions   |
    |                    |       |                      |
    |  JWT + RLS         |       |  Payments            |
    |  Role-based access |       |  Booking engine      |
    |  Session mgmt      |       |  Email service       |
    |                    |       |  CRM sync            |
    +--------------------+       |  Availability API    |
                                 +----------+-----------+
                                            |
              +-----------------------------+-----------------------------+
              |                             |                             |
    +---------v----------+       +----------v-----------+      +----------v-----------+
    |   PostgreSQL       |       |   Stripe             |      |   Resend → SES       |
    |                    |       |                      |      |                      |
    |  37 tables, RLS    |       |  6 subscription      |      |  10 transactional    |
    |  18 migrations     |       |  tiers               |      |  email templates     |
    |  Realtime PubSub   |       |  Webhook lifecycle   |      |  Coach + athlete     |
    +--------------------+       +----------------------+      |  notifications       |
                                                               +----------------------+
```

---

## What It Does

**Athlete Experience:** Sign up → onboarding flow → personalized dashboard with phase-based training (Foundation → Advanced), weekly drills, video submission via OnForm, community access, lesson booking with calendar + time slots, package/membership purchase via Stripe.

**Coach/Admin Experience:** Admin dashboard with revenue metrics, member management (profiles, roles, tiers), booking calendar with status management, curriculum CMS (levels → modules → lessons with video), event management (workshops, clinics, guest athletes), schedule/availability editor.

**Community:** Real-time posts, direct messaging, channels (Announcements, Q&A, Player Wins, Parents Room), reactions, polls, mentions, image upload, GIF picker. All powered by PostgreSQL realtime subscriptions.

---

## Technical Decisions

### Why Serverless-First
No idle compute. Edge functions handle spiky traffic (booking rushes, event registrations) without provisioning. Infrastructure cost at launch: **$30/month** serving paying customers. Projected cost at 500 users: **~$0.15/user/month**.

### Why PostgreSQL with RLS (Not DynamoDB)
Relational data model is natural for this domain — users have bookings, bookings reference service types, service types have pricing tiers. Row-Level Security eliminates an entire class of authorization bugs: the database enforces "users can only see their own data" at the query level, not the application level. 37 tables, all protected.

### Why Code Splitting Matters Here
41 page routes lazy-loaded via `React.lazy`. Parents checking their kid's training schedule shouldn't download the admin dashboard code. Initial bundle went from monolithic to route-level chunks — the booking page loads only what it needs.

### Why Stripe Webhooks (Not Polling)
Subscription state (active, past_due, canceled) is driven by Stripe webhooks hitting an edge function, which updates the database. The app never polls Stripe — it reads local state. This means zero Stripe API calls during normal user sessions, and subscription status is always consistent.

---

## Data Model (37 Tables)

```
Users & Auth              Bookings & Commerce       Content & Learning
──────────────            ──────────────────        ──────────────────
profiles                  bookings                  curriculum_levels
user_roles                service_types             curriculum_modules
                          coach_availability        lessons
Community                 blocked_times             lesson_completions
──────────────            packages                  drills
posts                     purchased_packages        drill_completions
comments                  user_packages             video_submissions
direct_messages
channels                  Events                    Gamification
channel_members           ──────                    ────────────
polls / poll_votes        events                    points
mentions / reactions      event_registrations       badges / user_badges
                          notifications
```

---

## Security

| Layer | Implementation |
|-------|---------------|
| **Data access** | Row-Level Security on all 37 tables |
| **Authentication** | JWT with automatic refresh, session persistence |
| **Authorization** | Role-based (`admin`, `coach`, `member`) checked at route and API level |
| **Secrets** | All API keys in encrypted vault — zero hardcoded credentials |
| **Input validation** | Zod schemas on all user-facing forms |
| **Error handling** | ErrorBoundary catches render failures; try/catch on all async operations |
| **CORS** | Restricted origins on all edge functions |

---

## Payments Architecture

```
User clicks "Subscribe" or "Book"
         |
         v
  create-checkout edge function
         |
         v
  Stripe Checkout Session (hosted)
         |
    [user pays]
         |
         v
  stripe-webhook edge function
         |
    ┌────┴────────────────────────┐
    |  checkout.session.completed |
    |  subscription.created      |
    |  subscription.updated      |
    |  subscription.deleted      |
    |  invoice.payment_succeeded |
    |  invoice.payment_failed    |
    └────┬────────────────────────┘
         |
         v
  Update profiles table
  (membership_tier, subscription_status,
   stripe_customer_id, credits)
         |
         v
  send-booking-confirmation
  (email to athlete + coach)
```

**6 membership tiers** from Community ($49/mo) to Hybrid Pro ($449/mo), each with different lesson rates, review quotas, and feature access.

---

## AWS Migration Path

Currently deployed on Amplify with Supabase backend. Migration to full AWS is planned and architected — each component has a direct replacement:

| Current | AWS Target | Trigger |
|---------|-----------|---------|
| Supabase Auth | Cognito (user pools, MFA) | Need SSO or compliance |
| Supabase PostgreSQL | Aurora Serverless v2 | Need scaling beyond Supabase limits |
| Supabase Edge Functions | Lambda + API Gateway | Need custom runtimes or VPC access |
| Resend | SES | Cost optimization at volume |
| Supabase Storage | S3 + CloudFront | Video library scaling |
| Supabase Realtime | API Gateway WebSocket + DynamoDB Streams | Need custom PubSub logic |

Migration order is designed to be incremental — swap one service at a time with zero downtime. Database migration via DMS with continuous replication.

---

## Stack

```
Frontend     React 18 · TypeScript · Vite 5 · Tailwind CSS · Shadcn/UI
State        React Query · Context API · Supabase Realtime subscriptions
Auth         JWT · Row-Level Security · Role-based access control
Database     PostgreSQL · 37 tables · 18 migrations · RLS on all tables
Backend      9 Edge Functions (Deno runtime)
Payments     Stripe Checkout · Webhooks · Customer Portal · 6 tiers
Email        Resend · 10 transactional templates · Coach + athlete notifications
CRM          GoHighLevel · Contact sync · Booking sync · Quiz data sync
Analytics    Meta Pixel · Google Analytics · GTM · A/B testing
Hosting      AWS Amplify · CloudFront CDN · ACM SSL
```

---

## Cost at Scale

| Users | Monthly Cost | Per User |
|-------|-------------|----------|
| 0-50 | $30 | — |
| 100 | $45 | $0.45 |
| 500 | $75 | $0.15 |
| 500+ (full AWS) | $60-140 | $0.12-0.28 |

Industry SaaS infrastructure benchmark: $1-5/user/month. This architecture runs at **5-10x below benchmark**.

---

*Architecture documentation for a live production platform. Source code is in a private repository.*
