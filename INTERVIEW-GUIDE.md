# Interview Guide — How to Talk About Swing Institute

> For walking a hiring manager, engineering lead, or staff+ interviewer through this project. Optimized for Senior Solutions Architect / Senior Full-Stack / Staff roles.

## The 30-second pitch

> "Swing Institute is a production baseball-training SaaS I built solo — web, iOS App Store, and PWA off a single TypeScript codebase, with AWS Bedrock powering computer-vision swing analysis. It's live with real users (MLB players, college athletes, youth prospects), runs on about $100 a month of infra, and has a clean migration path to AWS-native when I hit scale. The interesting engineering is in the AI pipeline, the payment and payout flow, and the three-channel notification system — all of which I can walk you through."

Use this verbatim as your opener when a recruiter or interviewer says "tell me about a project."

---

## The one-slide architecture story

When asked to whiteboard or share your screen:

1. **Clients.** Web + iOS + PWA, one TypeScript codebase. Capacitor wraps the web build into the iOS shell; platform-specific capabilities live behind a single adapter (Apple Vision pose, Sign in with Apple, APNs).
2. **Edge.** 35 Deno edge functions on Supabase. No long-running app server; cold start is ~100ms.
3. **Data.** Postgres with Row-Level Security on every user-scoped table. Service-role key never leaves the edge tier.
4. **External.** AWS Bedrock for vision AI. AWS SES for email. APNs direct for push. Stripe + Stripe Connect for money. Daily.co for live video. GoHighLevel for CRM sync.
5. **Observability.** Sentry for errors. A custom launch-readiness dashboard with 13 automated checks as a single pane of glass.

Pace yourself — one minute per layer is plenty.

---

## Senior-engineer answers to likely questions

### "Walk me through the most technically interesting piece."

> "The AI swing analysis pipeline, because it's genuinely hybrid cloud + on-device. The user records a swing on their phone. On iOS, a Swift Capacitor plugin I wrote uses Apple's `VNDetectHumanBodyPoseRequest` — runs on the Neural Engine, nearly free on battery. On web, the same shape comes from MediaPipe Tasks Vision in the browser. Both normalize to the same 33-landmark layout, so downstream code has one path.
>
> Client picks 8–12 key frames — stance, load, contact, follow-through — and only those get uploaded. That's the cost-control decision: Bedrock charges per image, so we send images, not video.
>
> The edge function verifies the JWT, checks clip ownership, then atomically decrements the user's AI-analysis credits via a Postgres RPC. I did that as an RPC specifically to avoid a TOCTOU race — if someone fires two uploads simultaneously from iPhone and iPad, only one gets the credit.
>
> Credits consumed, we sign a Bedrock call with AWS SIG V4 — same IAM credentials we use for SES — and ask Claude Sonnet 4 Vision for a structured biomechanical analysis. The system prompt encodes MLB benchmarks — hip-shoulder separation by age bracket, attack angle ranges, front-knee angles — so the output is repeatable, not freeform. We get back JSON, persist it to the clip, roll up the session average, return to the client. Total round trip around 8 seconds."

**Follow-ups to be ready for:**
- "Why not train a custom model?" → ADR-003. Cost + time; no data to train on until we have users.
- "What happens if Bedrock returns malformed JSON?" → We 500. Credit stays decremented because Bedrock billed. We accept that as a reasonable failure mode at our volume.
- "What about latency variance?" → UI shows an analyzing state. Realtime subscription updates when the clip's `ai_score` field flips. User perceives it as progress, not a blocked call.

---

### "How do you handle payments and payouts?"

> "Stripe for charges, Stripe Connect Express for coach payouts. The rule I enforce everywhere: the Stripe webhook is the only code path that marks anything paid. The browser-side success page is UX — it subscribes to the DB row flipping, but it doesn't write.
>
> Coach payouts are a weekly cron job. pg_cron fires `process-coach-payouts`, which queries earnings where `stripe_transfer_id IS NULL` and creates Stripe Transfers in batch. That predicate makes the job idempotent — if it partially fails, the next run picks up what's left. No double-pays.
>
> On top of that, there's a daily reconciliation job that walks Stripe's event stream and repairs any drift — which happens rarely but does happen when webhooks silently drop. Belt and suspenders."

**Follow-ups:**
- "How do you verify the webhook?" → Signature header, standard Stripe pattern, done at the top of the handler.
- "What's the failure mode if the reconciliation job finds drift?" → Logs + admin email alert; rarely need to act manually.
- "Why pg_cron and not a queue?" → ADR-014. Volume doesn't justify SQS + Lambda yet; idempotent cron is simpler.

---

### "How does authorization work?"

> "Three layers. First, Supabase Auth issues the JWT — that's identity. Second, a `user_roles` table — `admin`, `coach`, `parent`, `player` — governs authorization; it has an admin-only mutation path, so escalation isn't a client-side trick. Third, Row-Level Security on every user-scoped table enforces the tenancy boundary in the database. I don't rely on the application layer to keep users in their lane — the database refuses to return rows they can't see.
>
> On top of that, every sensitive edge function does explicit ownership checks — 'does this JWT's `user.id` match the resource's `user_id`?' — before acting. Defense in depth."

---

### "What broke in production, and what did you do about it?"

> "Push notifications, twice. Capacitor's push plugin is order-sensitive — listeners must be attached before `PushNotifications.register()` or the first token event is missed silently. Got bitten by this; added a comment, then a protected-surface marker in `CLAUDE.md` that any contributor — human or AI assistant — reads at session start. It's a lightweight CODEOWNERS-without-the-ceremony.
>
> Second: APNs tokens drift between sandbox and production builds. Users install the App Store build after having a dev build, the token changes silently, deliveries 404. Fix was in the `send-push` function — try sandbox first, retry production on failure, and delete the dead token if both fail. Delivery telemetry went to ~100% after that change."

This answer demonstrates: production debugging, change-control discipline, observability thinking, and that you learn from incidents.

---

### "How would this scale?"

> "Today's infra is about $100/month at < 500 MAU. The first bottleneck I'd hit is Postgres connections if the coach marketplace gets popular — Supabase's pooled connection limit is real. I'd move to Aurora PostgreSQL Serverless v2 when that approaches, keeping the same SQL so the migration is logical replication via AWS DMS.
>
> Edge functions can move to Lambda behind API Gateway when I want more control over cold starts and concurrency — the code is already standard Deno/TypeScript, so porting is mostly packaging.
>
> Storage is already S3-semantic via Supabase; that migration is `aws s3 sync` level of effort.
>
> The order matters: frontend first (free and easy), then auth (the riskiest cutover), then data (longest), then functions. I wrote this down in `doAWS-DEPLOYMENT-PLAN.md` so I'm not figuring it out during an emergency."

---

### "Tell me about a trade-off you're not fully happy with."

> "TypeScript strictness. I run loose strict mode — strict, but with escape hatches at integration boundaries where Supabase's generated types lag the schema. For MVP velocity, that's the right call — you still catch 80% of type bugs. But I know there's a class of runtime issues that only full-strict catches, and I'm paying for it with occasional Sentry reports on shape mismatches. Plan is to tighten module by module post-launch, starting with the payment and credit paths, which are the scariest to get wrong."

Demonstrates: awareness of trade-offs, a concrete remediation plan, not performative humility.

---

### "Why Supabase and not AWS-native from day one?"

> "Time to market. A solo engineer shipping Cognito + Aurora + Lambda + API Gateway equivalents from scratch is a month of infrastructure before the first page of product. Supabase gives me Postgres-with-RLS, auth, file storage, realtime, and edge functions in an afternoon. And because it's thin over Postgres, the eject path is SQL-portable — my migrations run on any Postgres.
>
> I have the migration plan written. It's phased — frontend first, auth second, data third — with AWS DMS doing logical replication during the data cutover. I'd start that migration at ~500 MAU or if a compliance requirement (SOC 2, data residency) forces it."

---

### "How do you test this?"

> "Playwright E2E for the critical user flows — the four partner application submits. Vitest for unit tests where I've got pure-function logic (rules engine scoring, entitlement math). Manual QA through the TierSwitcher admin tool for membership-gated paths. And the launch-readiness dashboard catches the integration-level regressions — if the mutual-confirm flow breaks, the 'at least 1 session mutually confirmed' check goes red.
>
> That's not a complete answer for a Fortune 500 — I'd add contract tests against Stripe, chaos tests on the 3-channel notification fan-out, and synthetic monitoring on the AI pipeline. For a single-founder pre-launch project, the ROI on those isn't there yet."

---

### "What would you do differently if you started over?"

> "Two things. One: I'd introduce the rules-engine scoring path before the LLM path, not alongside it. I did them in parallel and the LLM got priority because it's more impressive; what users actually needed first was instant feedback. Rules engine gets a user a score in 200ms with no cost.
>
> Two: I'd write ADRs contemporaneously from day one. I backfilled them at the portfolio stage and had to reconstruct some reasoning. ADRs are cheap insurance against future-me not remembering why past-me chose Capacitor over React Native."

Shows: reflection, process maturity, that you've actually thought about this.

---

## If they challenge you technically

### "This sounds like a lot of vendors."

Acknowledge → frame → land.
> "Yep — Supabase, AWS, Stripe, Daily.co, GoHighLevel, Sentry. Every one is an intentional choice where I picked the best primitive for that domain rather than least-common-denominator. Supabase for OLTP, AWS for heavy-lift (Bedrock, SES, CDN), Stripe for money, Daily.co for WebRTC — those are where the specialist is actually worth it. I have the vendor-reduction plan in `doAWS-DEPLOYMENT-PLAN.md` — I can collapse most of Supabase into AWS native when scale makes that cheaper."

### "Why didn't you use [Next.js / tRPC / Prisma / X]?"

Frame as deliberate.
> "For [Next.js]: SPA with client-side routing serves the product better than SSR — we're app-shaped, not content-shaped, and the marketing pages are fine as static. For [tRPC]: I wanted function boundaries on the network that I could rate-limit, auth at, and migrate to Lambda later — tRPC blurs that. For [Prisma]: I use SQL directly and generated Supabase types; Prisma's abstraction doesn't pay off when RLS is doing the heavy lifting. Nothing ideological, just trade-offs I evaluated."

### "How do you know the AI output is actually correct?"

Honest answer.
> "I don't, not in a statistical sense. I have the MLB benchmarks in the system prompt, and a rules engine that scores the same swings deterministically — so I can sanity-check the LLM against the rules engine and flag divergence. The real validation is our coaches reviewing the output in the CoachReviewInterface — they can override the AI, which feeds a dataset I'd use for fine-tuning or eval when we have volume. Today it's directionally useful; at 10x the users it becomes statistically validated."

### "Is Bedrock overkill? Could you do this with regex on landmark coords?"

> "For the category scores, yes — that's exactly what the rules engine does. The LLM earns its cost on the feedback prose, the drill recommendations, and the pro-comparison — the things that need language, not just numbers. It's why both paths exist: deterministic scoring is the baseline, LLM enrichment is the premium tier."

---

## Things to emphasize (they signal seniority)

- **You understand the difference between identity and authorization.** Most junior engineers conflate them.
- **You talk about the webhook as source of truth.** It's a telltale of someone who's actually shipped payments.
- **You name the TOCTOU race before being prompted.** Shows concurrency awareness.
- **You reference idempotency by predicate** (`WHERE stripe_transfer_id IS NULL`). Senior pattern.
- **You have a written migration plan.** Shows you think in phases, not in blast radius.
- **You use the word "trade-off" without hedging.** Seniors commit to choices and own the downside.

## Things to avoid

- **Don't say "it's basically a CRUD app."** It isn't, and that sells the project short.
- **Don't over-apologize for vendor choices.** Every vendor in this stack is an industry standard.
- **Don't rattle off every feature.** Pick three, go deep. Breadth is in the docs.
- **Don't say "AI wrote this."** Even if a tool helped, what matters in an interview is that you can explain any line of it and ship changes against it. You can.
- **Don't promise scale you haven't measured.** "I'd do X at 10k users" is honest; "this scales to millions" without load data is a red flag.

---

## The written artifact trail (lead with these links)

When you share the repo with a hiring manager or recruiter, send this short list:
- **Public architecture repo:** <https://github.com/jashabalcom/swing-institute-architecture>
- **This doc set:** `.` in the main repo
- **Live app:** <https://www.swinginstitutebaseball.com>

In that order. The public architecture repo lets them grok the system without asking for access to the private codebase. The portfolio docs show depth. The live app shows it actually works.

---

## Three interview formats, three openings

**Casual "tell me about a project":**
> "Want to hear about the baseball training SaaS I built solo? It's live, has real users, and has some genuinely interesting architecture I'd enjoy walking through."

**Formal behavioral round:**
> "I'll use Swing Institute — it's a production system I built end-to-end, so I can speak to the full lifecycle: product decisions, architecture trade-offs, shipping discipline, and post-launch operations."

**Deep-dive technical round:**
> "I'd like to walk through the AI swing analysis pipeline — there's a hybrid on-device + cloud design, a non-trivial cost-control decision, and a concurrency race I had to handle carefully. Should cover most of what you'd want to evaluate."

---

## Last thing — keep it grounded

You built a real platform. It has real users. You made real trade-offs under real constraints. The interview isn't a test of whether you know CAP theorem cold; it's a test of whether you can make good calls with incomplete information and explain them clearly.

Everything in this guide is true about this codebase. Don't inflate, don't hedge, don't pretend. "This is what we shipped, this is why, here's where it can go next" is the right posture from a senior.
