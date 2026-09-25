# Build Timeline — Stage by Stage

> What shipped, in roughly the order it shipped, with the reasoning behind each stage. Sourced from the full history of `main`, which begins in April 2026.

This is not a changelog. It's a product-and-architecture narrative — what problem was being solved at each stage and what decision followed.

---

## Stage 0 — Foundation (auth, data model, first pages)

**Goal.** Bootstrap a multi-surface product: web-first, with the bones to add iOS later.

**Shipped.**
- Supabase project provisioned; initial migrations for `profiles`, `user_roles`, `auth` integration.
- Vite + React + TypeScript + shadcn/ui scaffolding.
- Auth flow: email/password + Apple Sign In on iOS (via Supabase Auth).
- Route guards: `<ProtectedRoute>`, `<AdminRoute>`, `<CoachRoute>`.
- `AuthContext` with session + profile hydration and Capacitor `Preferences` bridge for native-safe session storage.

**Decision.** Postgres + RLS at the DB layer (not ORM-level auth). Every user-scoped table ships with an RLS policy from day one — retrofitting RLS later is painful.

**Commit landmarks.** `Launch-ready platform with AWS deployment`, `ios: tighten Apple compliance for App Store submission`.

---

## Stage 1 — Monetization & membership tiers

**Goal.** Ship paid access with clear tier boundaries.

**Shipped.**
- Stripe Checkout integration, subscription webhooks, customer portal.
- 7+ membership tiers (Free, Online, Online Plus, Online Pro, Hybrid, Group 2x, Group Unlimited).
- Entitlement system via `useEntitlement` hook; tier-based feature gating.
- Multi-step cancellation flow with downsell retention.
- Admin `TierSwitcher` for QA.

**Decision.** Webhook-as-source-of-truth from the start (see [ADR-006](ENGINEERING-DECISIONS.md#adr-006--webhook-as-source-of-truth-for-stripe-with-a-reconciliation-backstop)) — the client-side success screen is UX-only; the DB is written by the Stripe webhook.

**Commit landmarks.** `Payment automation: access gating, dunning emails, admin quick charge`, `Fix critical stripe-webhook bugs — user lookup, account creation, password setup`.

---

## Stage 2 — Coaching infrastructure & booking

**Goal.** Enable 1-on-1 sessions (virtual and in-person) bookable against a coach's calendar.

**Shipped.**
- Coach profiles + availability editor.
- Service types, duration + pricing metadata, admin CRUD.
- Booking flow with credit-based deduction, Stripe fallback for non-credit tiers.
- Cal.com-style premium calendar redesign with month grid + availability dots.
- Mutual check-in (coach + player) with GPS for in-person verification.
- Session reminder cron jobs (24h, 2h).

**Decision.** Atomic credit RPC (`decrement_hybrid_credit`, `decrement_group_credit`) instead of read-then-write — no TOCTOU races on concurrent bookings ([ADR-005](ENGINEERING-DECISIONS.md#adr-005--atomic-credit-consumption-via-postgres-rpc-never-read-then-write)).

**Commit landmarks.** `Multi-coach admin, contact management, coach scheduling`, `feat(booking): premium Cal.com-style redesign + fix checkout bug`, `Fix: credit restoration on cancellation + past-date validation`.

---

## Stage 3 — Academy (curriculum + progress tracking)

**Goal.** On-demand video training library with structured learning paths.

**Shipped.**
- Three-level hierarchy: Levels → Modules → Lessons.
- Video uploader, lesson editor, per-level gating.
- Progress tracking per user per lesson.
- Drill-of-the-week surfacing in community.
- Admin curriculum builder pages.

**Decision.** Content lives in Postgres (not a CMS) with Supabase Storage for video blobs — keeps auth model unified and RLS in force on curriculum access.

---

## Stage 4 — Community (social layer)

**Goal.** Make the platform sticky beyond the one-to-one coaching relationship.

**Shipped.**
- Posts, comments, reactions, mentions, polls.
- Direct messages + group chats with typing indicator, read receipts, message grouping.
- Notifications bell with real-time subscription.
- Moderation queue, block/report, content filter.
- Member directory, online-now widget, activity badges.
- Tier badges + tier-colored avatar rings.

**Decision.** Supabase Realtime for presence and notifications instead of adding a queue system (Redis/Pusher). Native capability, no added infra.

**Commit landmarks.** `Community Phase A: new components, sidebar polish, drill feature, presence`, `Community Phase A: DM read receipts, typing indicator, message grouping`.

---

## Stage 5 — Rules-based scoring engine (first AI surface)

**Goal.** Give every user instant post-swing feedback without waiting on an LLM.

**Shipped.**
- Browser-side pose detection via MediaPipe Tasks Vision.
- Rules engine in TypeScript: age-calibrated ranges, per-category scoring (stance, load, contact, follow-through).
- Instant score display in the swing recorder.
- Scores persist to `swing_clips` on detection.
- Age group field on profile to calibrate thresholds.

**Decision.** Ship rules engine first, then LLM. The rules engine is deterministic, cheap, and educational — even when the LLM is offline or the user is out of credits, they still get a score.

**Commit landmarks.** `feat: add rules-based swing scoring engine with age-calibrated ranges`, `feat: wire rules engine scoring into swing detection with instant score display`.

---

## Stage 6 — AI swing analysis (AWS Bedrock, Claude Sonnet 4 vision)

**Goal.** Deep biomechanical feedback from a real vision model.

**Shipped.**
- `analyze-swing` edge function with AWS SIG V4 signing via `aws4fetch`.
- 8–12 key-frame extraction client-side (stance → follow-through).
- Structured system prompt with MLB benchmark tables (hip-shoulder separation, attack angle, front knee angle, etc.).
- Strict JSON output contract: 5 category scores, 20+ metrics, drill recommendations, pro comparison.
- Credit-gated per tier; staff (admin/coach) bypass.
- Session `avg_ai_score` rolling update.
- Result surfaces on dashboard score card, progress chart, smart-drills card.

**Decision.** Bedrock over direct Anthropic API ([ADR-003](ENGINEERING-DECISIONS.md#adr-003--aws-bedrock-for-vision-ai-instead-of-self-hosting-a-pose-evaluation-model)) — single AWS bill, IAM-based auth, same credentials as SES.

**Commit landmarks.** `feat: save AI scores to swing_clips on detection`, `feat: add LatestScoreCard component with score ring and category bars`, `feat: Enhanced Dashboard — Score Card, Progress Chart, Smart Drills`.

---

## Stage 7 — iOS native (Capacitor + Apple Vision + APNs)

**Goal.** App Store presence with native performance where it matters.

**Shipped.**
- Capacitor 8 wrap around the same Vite build.
- Swift `PoseDetectorPlugin` using `VNDetectHumanBodyPoseRequest`, mapped to MediaPipe-compatible 33-point layout.
- Swift `AppleAuthPlugin` for Sign in with Apple.
- APNs direct integration with dual sandbox/prod endpoint retry + stale token cleanup.
- `usePushNotifications` hook with listener-before-register ordering (a must-get-right Capacitor pitfall).
- `Capacitor.Preferences` bridge for auth session storage.
- Platform guards throughout (`Capacitor.isNativePlatform()`).
- iOS splash → React-paint handoff to eliminate white flash.
- Native-first intro carousel + dedicated native welcome screen.
- Safe-area + notch handling, premium mobile pass.

**Decision.** Native Swift plugins where they earn their keep (battery, performance), JS everywhere else ([ADR-004](ENGINEERING-DECISIONS.md#adr-004--ios-native-apple-vision-pose-plugin-mediapipe-on-web-shared-data-shape)).

**Commit landmarks.** `Bump iOS deployment target to 16.4 + add encryption declaration`, `fix iOS long-press + batch selection tap bug`, `feat(ios): first-time NativeIntro carousel`.

---

## Stage 8 — 3-channel notifications (push + email + in-app)

**Goal.** Reliable delivery of every user-facing event.

**Shipped.**
- `send-push` edge function with dual APNs endpoint retry + token cleanup.
- `send-email` edge function (3000+ lines, 80+ templates, 15 template-code families) using AWS SES + SIG V4.
- Shared `email-styles.ts` design token set (navy + sky-blue brand).
- `notifications` table + Realtime subscription → bell icon badge.
- Admin email alerts (new registration, flagged content, important activity).
- Protected-surface markers in `CLAUDE.md` for every file in the notification path.

**Decision.** AWS SES over Resend ([ADR-013](ENGINEERING-DECISIONS.md#adr-013--aws-ses-over-resendsendgrid-for-transactional-email)) — single AWS bill, same IAM creds as Bedrock, 5x cheaper at volume.

**Commit landmarks.** `Add W1 welcome email + admin alerts on new member signup`, `Add comprehensive notifications for all Stripe webhook events`, `Add error logging to push notification registration`.

---

## Stage 9 — Coach payouts (Stripe Connect Express + weekly cron)

**Goal.** End-to-end money movement — player pays → coach gets paid — without a payments engineer.

**Shipped.**
- Stripe Connect Express onboarding flow via `create-connect-account` edge function.
- Earnings recorded per confirmed session with commission splits.
- Weekly `process-coach-payouts` edge function triggered by pg_cron.
- `coach-weekly-payout` aggregates unpaid earnings → creates Stripe Transfers → persists `stripe_transfer_id`.
- `AdminPayroll` dashboard for reconciliation.
- `stripe-reconciliation` daily sweep to repair drift.

**Decision.** Scheduled job idempotent on `stripe_transfer_id IS NULL` — safe to rerun if a cron execution partially fails.

---

## Stage 10 — Partner programs (Coach / NIL / Ambassador / Affiliate)

**Goal.** Recruit supply-side (coaches) and growth partners.

**Shipped.**
- 4 program landing pages with variant-driven shared components (`LandingHero`, `ValueStack`, `TierCard`, `MultiStepForm`, `IncomeCalculator`).
- `partner_applications` table with `program` discriminator (single table, 4 programs) ([ADR-008](ENGINEERING-DECISIONS.md#adr-008--single-partner_applications-table-with-a-program-discriminator)).
- `submit-partner-application` edge function with program-specific Zod validation.
- Intro-video storage bucket with 100 MB limit and RLS.
- `AdminPartnerApplications` review queue spanning all 4 programs.
- Ambassador / NIL / Affiliate / Coach dashboards for accepted partners.
- Referral attribution + commission tracking.
- FTC-safe disclaimers on every page (NIL compliance).
- Playwright E2E coverage on all 4 submit flows.

**Decision.** Extract a shared `landing-premium/` component library instead of 4 page-by-page duplicates — 40% less code, easier to evolve.

**Commit landmarks.** `milestone: landing pages shipped (4 pages + admin queue + e2e + edge fn live)`, `feat(landing-premium): extract MultiStepForm generic wizard with progress bar`.

---

## Stage 11 — Landing V2 premium redesign + app CTA

**Goal.** Conversion-grade marketing surface with App Store call-to-action.

**Shipped.**
- Premium redesign across all 4 landing pages to match home-page design language.
- `AppDownloadCTA` with iPhone-17-Pro mockup, waitlist mode, QR code.
- Carousel of real app screenshots (Home / Daily Challenge / AI Analysis / Swing Vault).
- Scroll-triggered tilt-to-straight phone motion (Apple product-page feel).
- Sticky mobile apply bar on partner pages.
- Cedric Mullins (MLB All-Star) video posters + metadata preload for instant thumbnails.
- V2 pricing page with segmented toggle + geo-aware In-Person flow + city waitlist.

---

## Stage 12 — COPPA + parent dashboard

**Goal.** Legal-compliant multi-generational access (parents managing youth athletes).

**Shipped.**
- Age gate on signup.
- COPPA-compliant parent verification flow (`send-coppa-verification`).
- Parent creates child accounts with auto-onboarding (`create-child-account`).
- Parent dashboard with multi-child roster + session snapshots.
- Parent-side email family (PS templates).

---

## Stage 13 — Group classes (Sprints 1–3)

**Goal.** Many-to-one training sessions (camps, clinics).

**Shipped.**
- Group class foundation (admin UI + schema) — Sprint 1.
- Customer browse / detail / registration / my-classes — Sprint 2.
- Coach self-service + notifications + ratings — Sprint 3.
- Capacity + waitlist promotion via `notify-waitlist-promoted`.

---

## Stage 14 — Swing Vault UX overhaul

**Goal.** Make a user's growing library of swings genuinely browsable.

**Shipped.**
- Session cards, filters, context menu, batch mode.
- Two-tier thumbnail generation (server-side + remote-URL fallback).
- Share + social system (Web Share API, referral codes, shared-clip pages).
- Latest-first default sort.
- AI scores + category indicators on vault clip cards.

---

## Stage 15 — Coach review queue (desktop power mode)

**Goal.** Let coaches efficiently clear a queue of player submissions.

**Shipped.**
- `ReviewQueue` component with priority sorting + urgency filters.
- Keyboard shortcuts hook for rapid navigation.
- `AIPreAnalysis` card showing rules engine scores for coach context.
- Skip button, auto-advance.
- Coach Dashboard wiring.

---

## Stage 16 — Admin surface maturation

**Goal.** Operationally self-serve: members, coaches, schedule, payroll, revenue, moderation.

**Shipped (20+ admin pages).**
- AdminDashboard, AdminMembers, AdminCoaches, AdminCoachMarketplace, AdminCoachAssignments.
- AdminAcademy (curriculum builder), AdminDrills, AdminVideos.
- AdminSchedule with drag-drop grid, AdminClasses, AdminEvents.
- AdminPackages, AdminServiceTypes, AdminRevenue (charts), AdminPayroll.
- AdminReferrals, AdminModeration, AdminTeams.
- AdminPartnerApplications (cross-program queue).
- AdminCityWaitlist.

---

## Stage 17 — Launch readiness dashboard

**Goal.** Single pane of glass for go/no-go launch decisions.

**Shipped.**
- `AdminLaunchReadiness` page with 13 automated checks across critical / health / data categories.
- `launch_readiness_rpc` aggregates stats in one DB round trip.
- pg_cron last-run timestamps surfaced inline.
- Pass/warn/fail status with inline remediation guidance for each check.

**Decision.** A custom dashboard beats an external ops tool at this stage — founder-runnable, in-app, zero credential sprawl ([ADR-010](ENGINEERING-DECISIONS.md#adr-010--launch-readiness-dashboard-as-a-first-class-page)).

**Commit landmarks.** `feat(admin): launch-readiness dashboard — one page, 13 automated checks`.

---

## Stage 18 — Swing V2 (planned next)

**Design spec shipped** (`docs/SWING-V2-SPEC.md` referenced in commit).

**Planned.**
- User-armed capture (user explicitly indicates "I'm about to swing") to eliminate false-trigger noise.
- Deterministic scoring path as the default; Bedrock-enhanced path as a tier perk.
- iOS 17 3D pose upgrade (`VNDetectHuman3DBodyPoseRequest`) for better depth-aware biomechanics.
- 21 TDD tasks across 8 phases (foundation → launch).

**Commit landmarks.** `docs(swing): V2 design spec — user-armed capture, deterministic scoring, iOS 17 3D pose upgrade`, `docs(swing): V1 implementation plan — 21 TDD tasks across 8 phases`.

---

## Stage 19 — Marketplace lifecycle, growth, and CI that actually gates (Aug 2026)

**Goal.** Close the gap between "the feature exists" and "the business process completes without me."

**Shipped.**
- Marketplace lifecycle: booking-request acceptance windows, auto-cancel on expiry, en-route + photo check-in, dispute SLA resolution, reviews and coach reputation, marketplace-specific payment reconciliation.
- ~15 `pg_cron` sweepers for every process that can stall ([ADR-022](ENGINEERING-DECISIONS.md#adr-022--sweepers-over-queues-for-stalled-business-processes)).
- Trust & safety: Checkr background checks, Stripe Identity verification, coach certification, report + block on every UGC surface, DM filtering.
- Growth: Meta Conversions API, referral attribution and commission, affiliate invites, lead cadence, per-city waitlists.
- Email v3 redesign (navy + red, Oswald display, one CTA) across the 12 functions that bundle `email-styles.ts`.
- `canary-runner` — six synthetic checks every 5 minutes, plus an **external dead-man's switch** so the absence of a signal pages the operator when the whole project goes dark.

**The uncomfortable finding.** The 1,849 frontend tests had been running **nowhere** in CI. The quality job went straight from the lint ratchet to the Deno gates, and the only other job was `continue-on-error`. Every vitest file in `src/` was decoration — writing a test proved nothing about a PR. Wiring it in was a two-line change; it was red when it landed.

**Decision.** Gate on ratchets rather than thresholds ([ADR-016](ENGINEERING-DECISIONS.md#adr-016--ratchet-based-quality-gates-instead-of-hard-thresholds)) — a permanently-red gate is one everyone learns to ignore, which is worse than none.

**Commit landmarks.** `fix(ci): the test gate had never actually run, and it was red`, `feat(monitoring): external dead-man's switch on the canary`.

---

## Stage 20 — Launch audit and the security hardening it forced (Aug 15–22, 2026)

**Goal.** Find out what was actually wrong before customers did.

**Method.** Nine dimensions audited in parallel against the repo *and* live production, every finding then handed to a second reviewer whose only job was to **refute** it ([ADR-020](ENGINEERING-DECISIONS.md#adr-020--adversarial-verification-for-audit-findings)).

**Result.** 89 findings raised, **36 confirmed**, **14 actively refuted** — a 16% refutation rate. Two agents died on an expired OAuth token, so 5 funnel findings were never verified and are labeled *raised*, not *confirmed*. 16 findings closed within the first day.

**The blocker.** A small number of `SECURITY DEFINER` credit RPCs were reachable by the anonymous role. The guard pattern short-circuits on a NULL uid by design — safe only because that role is revoked at the grant level, which these had never received. Closed before launch and verified at the privilege level afterward. *(Specifics are held in the private repo.)* See [ADR-018](ENGINEERING-DECISIONS.md#adr-018--grant-level-lockdown-as-a-distinct-security-layer-above-rls).

**The two-stage fix.** Any authenticated user could read every column of every profile, including minors' legal names and parent contact details. Revoking the grant outright breaks login for 54 frontend read sites, so it shipped in two stages — migrate the reads onto a narrowed `member_directory` view first, revoke only once that build was live ([ADR-019](ENGINEERING-DECISIONS.md#adr-019--two-stage-rollout-for-permission-narrowing-migrations)).

**Also closed.** A COPPA age-gate bypass in the parental-consent step; ad pixels firing on under-13 sessions; two open-redirect gaps; account deletion that deleted nothing; a refunded lesson cancelling the member's whole subscription; a failed booking that ate the credit.

**App Store compliance.** Guideline 3.1.1 anti-steering (no subscription price on native), Academy gated behind coming-soon, Sentry stopped recording minors, report + block mounted on every UGC surface.

**Commit landmarks.** `feat(security): APPLY stage B — the profiles read path is narrowed`, `fix(security): finish stage A — cross-user profile reads migrated to member_directory`, `fix(coppa): close the age-gate bypass in the parental-consent step`.

---

## Cross-cutting patterns observed across stages

Looking across the whole history, certain patterns repeat:

1. **Plan-first cadence.** Design specs and implementation plans land in `docs/` before the feature ships. See `docs/NEXT-SESSION-*.md`, `docs/PARENT-VIEW-SPEC.md`, `docs/DASHBOARD-ACADEMY-UPGRADE-SPEC.md`.
2. **Merge commits mark phase gates.** Every meaningful phase ends with a `Merge feat/*` commit, giving clean rollback points.
3. **Revert-on-doubt.** Experiments that didn't pan out get reverted cleanly (recent branding refresh rolled back via two reverts) — branches stay shippable.
4. **Post-feature hardening.** `CTO audit: security hardening, error visibility, and production optimization` is a deliberate audit commit pattern that surfaces regularly.
5. **Type hygiene sweeps.** `Regenerate Supabase types and remove 109 as-any casts` — periodic type debt payment rather than letting it compound.
6. **Protected-surfaces discipline.** When a system becomes production-critical (notifications, Stripe, credits), it gets marked PROTECTED in `CLAUDE.md` to signal change-control to future contributors.

---

## Reading this timeline

- **Volume is not the point; the sequence is.** Every stage follows one repeating shape: plan → build → harden → protect.
- **Each stage solves a concrete business problem.** Stages 5–6 ship an AI surface because users need instant feedback; stage 9 ships Stripe Connect because coaches need to get paid; stage 20 hardens security because an audit proved it was needed.
- **The reasoning lives next door.** See [ARCHITECTURE.md](ARCHITECTURE.md) for how each stage's pieces fit together, and [ENGINEERING-DECISIONS.md](ENGINEERING-DECISIONS.md) for why each choice was made over its alternatives.
