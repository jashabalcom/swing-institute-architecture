# Engineering Decisions — ADR Log

> Short-form architecture decision records. Each one names the problem, the choice, the real alternatives considered, and the trade-offs we accepted.

**Format:** every ADR is structured `Context → Decision → Alternatives → Trade-offs → Status`. That's the same shape AWS, Spotify, and most mature engineering orgs use. Keeping them short makes them readable; a wall of prose nobody reads is worse than no ADR at all.

---

## ADR-001 — Capacitor over React Native for the iOS app

**Context.** Business requires an iOS App Store presence alongside the web app. Team size: 1 primary engineer. Web product already built in React.

**Decision.** Wrap the existing Vite + React SPA with Capacitor 8 and ship one build to three surfaces (web, PWA, iOS).

**Alternatives considered.**
1. React Native — a parallel mobile codebase. Shared logic via a monorepo, but UI has to be re-implemented (View vs div, StyleSheet vs CSS).
2. Native Swift app consuming the same API. Best native feel. Requires a second engineer or 3–6 months of solo ramp.
3. PWA-only (no App Store presence).

**Trade-offs accepted.**
- **Pro:** single codebase, one deploy. Web changes land on iOS after `npm run build && npx cap sync ios`.
- **Pro:** native capability when needed — we still wrote real Swift for Apple Vision and Sign in with Apple as Capacitor plugins.
- **Con:** WebView performance vs native — mitigated by keeping the bundle lean (Vite code-splits 50+ chunks) and using `transform`-based animations only.
- **Con:** some iOS-specific UX (pull-to-refresh, haptics, status bar) requires platform guards — wrapped in `Capacitor.isNativePlatform()` checks.

**Status.** Accepted. The iOS app ships the same React screens with native pose detection and push. No evidence we would have shipped faster another way.

---

## ADR-002 — Supabase (Postgres + Edge Functions) over self-managed AWS at launch

**Context.** MVP needed auth, DB, file storage, serverless functions, and realtime. Budget < $100/mo. Single engineer; ops time is at a premium.

**Decision.** Run the backend on Supabase (managed Postgres + Deno Edge Functions + RLS + Storage + Realtime) with a documented migration path to AWS-native when scale demands it.

**Alternatives considered.**
1. **AWS-native from day 1** — Cognito + Aurora + Lambda + API Gateway + S3. Most flexible, highest long-term ceiling. ~2x the time-to-MVP for auth + RLS equivalents.
2. **Firebase** — faster but locks us into NoSQL and Google. Credit/payment correctness is harder without SQL transactions.
3. **AppWrite / Convex** — promising but smaller ecosystems and less production-hardened.

**Trade-offs accepted.**
- **Pro:** Postgres with RLS is the right primitive for a multi-tenant SaaS — tenancy is enforced at the DB, not the app.
- **Pro:** migrations are SQL, not framework magic. 89 forward-only files.
- **Pro:** edge functions are Deno (first-class TS, no compile step), which matches the rest of the codebase.
- **Con:** vendor concentration — if Supabase has an incident, we're dark. Mitigated by (a) the migration plan and (b) Supabase being a thin layer over standard Postgres, so DB and SQL portability are guaranteed.
- **Con:** edge function cold starts are non-zero (~100 ms) on low-traffic endpoints. Not a user-visible concern for our workloads.

**Status.** Accepted at launch scale. [docs/AWS-DEPLOYMENT-PLAN.md](AWS-DEPLOYMENT-PLAN.md) is the exit plan when we cross ~500 MAU or need compliance features (SOC 2, data residency).

---

## ADR-003 — AWS Bedrock for vision AI instead of self-hosting a pose-evaluation model

**Context.** Product needs MLB-caliber biomechanical analysis from a phone video. Options range from training our own CV model to calling a hosted LLM.

**Decision.** Use AWS Bedrock with Claude Sonnet 4 (vision-capable). Send 8–12 key frames as base64 JPEGs with a heavily-structured system prompt carrying MLB benchmarks. Parse structured JSON back.

**Alternatives considered.**
1. **Train a custom CV model.** 6–12 months of data labeling, infra, iteration. Can't justify before we have 1000 real users to learn from.
2. **Use MediaPipe outputs directly without an LLM.** MediaPipe gives landmarks, not coaching; we'd still need a rules engine to convert landmarks → feedback. That's a worse product and comparable engineering effort.
3. **OpenAI GPT-4o or Anthropic direct API.** Comparable capability, different procurement. Bedrock gives us a single AWS bill, IAM-based auth, and keeps us on a first-party AWS data path.
4. **Google Gemini.** Strong vision model, different vendor surface.

**Trade-offs accepted.**
- **Pro:** ships today. Real analyses from day one, real customer feedback loop.
- **Pro:** structured prompting with benchmark tables means the model's output is repeatable and gradeable, not freeform text.
- **Pro:** Bedrock → AWS SIG V4 auth means the same IAM credentials that sign SES requests sign inference requests.
- **Con:** per-analysis cost (~$0.02–0.05 depending on frame count). Mitigated by client-side frame extraction — we never send the full video.
- **Con:** coupling to a specific model version. Mitigated by persisting the raw `ai_analysis` JSONB blob so old scores remain interpretable if we upgrade models.

**Status.** Accepted. Credit-gated per tier ([src/hooks/useEntitlement.ts](https://github.com/jashabalcom/swinginsitute/blob/main/src/hooks/useEntitlement.ts)) so cost is predictable. Staff (admin + coach) bypass the gate for QA.

---

## ADR-004 — iOS native Apple Vision pose plugin, MediaPipe on web, shared data shape

**Context.** Client-side pose detection must run on both iOS and the web. MediaPipe works in the browser but is slow and battery-hungry on iOS WebViews. Apple Vision is native and nearly free on iOS 14+.

**Decision.** Write a Swift Capacitor plugin using `VNDetectHumanBodyPoseRequest` for iOS. On the web, use `@mediapipe/tasks-vision`. Normalize both outputs to MediaPipe's 33-landmark index layout so downstream TypeScript has a single path.

**Alternatives considered.**
1. **MediaPipe everywhere.** Simpler code, poor iOS performance.
2. **Apple Vision via WKWebView bridge hack.** Fragile, not a public API.
3. **Server-side pose detection.** Would require uploading the full video — expensive bandwidth, slower UX.

**Trade-offs accepted.**
- **Pro:** battery/thermal profile on iOS is dramatically better. Apple Vision runs on the Neural Engine.
- **Pro:** one TS `PoseLandmark[33]` shape. Web and iOS branches diverge at 1 module boundary (the plugin) and nowhere else.
- **Con:** Apple Vision has 19 joints; MediaPipe has 33. The Swift plugin fills the 14 unused slots with `{x:0,y:0,visibility:0}` so TS code filters them out via visibility checks. This is a documented compromise; no downstream code uses the missing points.
- **Con:** two code paths to maintain. Mitigated by a single adapter interface and a feature flag to force MediaPipe on iOS if needed for A/B testing.

**Status.** Accepted. See [ios/App/App/PoseDetectorPlugin.swift](https://github.com/jashabalcom/swinginsitute/blob/main/ios/App/App/PoseDetectorPlugin.swift) and [src/plugins/PoseDetector.ts](https://github.com/jashabalcom/swinginsitute/blob/main/src/plugins/PoseDetector.ts).

---

## ADR-005 — Atomic credit consumption via Postgres RPC, never read-then-write

**Context.** Users get N AI analyses per month based on their subscription tier. Concurrent uploads from the same user (iPhone + iPad, or a flaky retry) could race a naive `SELECT → check → UPDATE` into a free analysis.

**Decision.** Every limited resource decrement runs through a single-statement Postgres RPC:
```sql
UPDATE profiles
SET deep_ai_credits_remaining = deep_ai_credits_remaining - 1
WHERE user_id = p_user_id AND deep_ai_credits_remaining > 0
RETURNING deep_ai_credits_remaining;
```
Either you got the credit (row returned) or you didn't. No TOCTOU window.

**Alternatives considered.**
1. **Serializable transaction isolation.** Works but serializes all credit ops globally. Postgres retries become visible latency.
2. **Application-level lock (Redis + Redlock).** Adds a dependency and still risks a slow RPC seeing a stale value.
3. **Per-user advisory locks.** Good but more code than needed — the atomic UPDATE is already atomic.

**Trade-offs accepted.**
- **Pro:** one round-trip, impossible to race.
- **Pro:** same pattern for `decrement_hybrid_credit`, `decrement_group_credit`, `decrement_deep_ai_credit` — four RPCs, one mental model.
- **Con:** the RPC is DB-specific. If we migrate to DynamoDB, we need a different primitive (conditional write with `if_not_exists` predicate).

**Status.** Accepted. Marked PROTECTED in [CLAUDE.md](https://github.com/jashabalcom/swinginsitute) — any change goes through explicit review.

---

## ADR-006 — Webhook-as-source-of-truth for Stripe, with a reconciliation backstop

**Context.** Stripe Checkout redirects the user to a success URL on completion, but the user can close the tab, lose network, or take the path via Apple Pay that never returns. We need a consistent "did they pay?" answer.

**Decision.** The `stripe-webhook` edge function is the only code path that marks a booking paid. The browser-side "success" screen calls `verify-checkout` to handle the happy-path UI but never writes the paid state itself. A daily `stripe-reconciliation` job walks Stripe's recent events and repairs any drift.

**Alternatives considered.**
1. **Optimistic client write + webhook as backup.** Invites double-writes and inconsistency.
2. **Poll Stripe on page load.** Rate-limited, slow, racy.
3. **Webhook-only with no reconciliation.** Leaves silent delivery failures unbackstopped.

**Trade-offs accepted.**
- **Pro:** one writer, one source of truth. Disputes are easy to trace.
- **Pro:** webhook signature verification guarantees authenticity.
- **Con:** webhook latency (usually < 1s) means the user briefly sees "payment processing" before the DB row flips. Mitigated by the `verify-checkout` call and realtime subscription.

**Status.** Accepted. Webhook is marked PROTECTED.

---

## ADR-007 — Three-channel notification fan-out with self-healing APNs

**Context.** Users expect push, email, and in-app notifications for important events. Any of the three can silently fail (APNs token drift, SES suppression list, Realtime reconnect).

**Decision.** A single edge function emits to all three channels in parallel via `Promise.allSettled`. APNs delivery retries both sandbox and production endpoints and deletes stale tokens on failure.

**Alternatives considered.**
1. **Route through a unified provider (OneSignal, Pusher).** Adds cost + vendor; loses the direct APNs token lifecycle control.
2. **Email-only until scale justifies push.** Unacceptable UX for real-time events (DMs, coach feedback).
3. **Serial delivery (push → email → in-app).** A hang on one channel blocks the others.

**Trade-offs accepted.**
- **Pro:** ~100% delivery in telemetry. Observed directly in the launch-readiness dashboard.
- **Pro:** no SaaS middleman, full control over token and template lifecycle.
- **Con:** we own more code. Mitigated by the protected-file markers and a single well-tested path.

**Status.** Accepted. Marked PROTECTED in [CLAUDE.md](https://github.com/jashabalcom/swinginsitute).

---

## ADR-008 — Single `partner_applications` table with a program discriminator

**Context.** Four partner programs (Coach / NIL / Ambassador / Affiliate) collect overlapping-but-not-identical data. Admin needs a unified review queue.

**Decision.** One table, a `program` enum column, JSONB `extra_data` for program-specific fields. A single RLS policy, a single admin UI ([AdminPartnerApplications.tsx](https://github.com/jashabalcom/swinginsitute/blob/main/src/pages/AdminPartnerApplications.tsx)).

**Alternatives considered.**
1. **Four parallel tables.** 4x the RLS policies, 4x the admin queries, harder to cross-reference a person who applies to multiple programs.
2. **Polymorphic via foreign keys to per-program tables.** More joins, less flexibility.

**Trade-offs accepted.**
- **Pro:** one migration, one RLS policy, one review queue.
- **Pro:** easy to add a fifth program without schema change.
- **Con:** JSONB fields aren't typed at the DB layer. Mitigated by Zod validation at the edge function boundary and TS types in the frontend.

**Status.** Accepted.

---

## ADR-009 — Protected-surfaces convention via `CLAUDE.md` markers

**Context.** Production-verified code (notification pipeline, credit RPCs, Stripe webhook, SES config) was broken multiple times by well-intentioned refactors that didn't understand the runtime constraints (listener ordering, upsert vs delete-insert, signature verification).

**Decision.** Mark the files in `CLAUDE.md` with a PROTECTED header explaining why they're sensitive and what breaks if changed. Every maintainer (human or AI) reads `CLAUDE.md` as part of their onboarding.

**Alternatives considered.**
1. **CODEOWNERS + PR review gate.** Strong but doesn't prevent the refactor from being proposed in the first place.
2. **Runtime assertions.** Possible for some (verify signature) but not others (listener-order bugs are silent).
3. **Inline comments only.** Not durable; comments get deleted during refactors.

**Trade-offs accepted.**
- **Pro:** a single discoverable location for "don't touch this without reading why."
- **Pro:** readable by humans and AI assistants alike — the AI-collab workflow reads `CLAUDE.md` on every session start.
- **Con:** depends on discipline. Mitigated by keeping the list short (4 surfaces).

**Status.** Accepted. See [CLAUDE.md](https://github.com/jashabalcom/swinginsitute) "PROTECTED" sections.

---

## ADR-010 — Launch-readiness dashboard as a first-class page

**Context.** Before a go-live event, we need to verify ~13 things across Stripe Connect, session flows, pg_cron, push tokens, and data integrity. "Run some SQL and see" does not scale and is error-prone under pressure.

**Decision.** Build [AdminLaunchReadiness.tsx](https://github.com/jashabalcom/swinginsitute/blob/main/src/pages/AdminLaunchReadiness.tsx) as a single page with 13 automated checks (pass/warn/fail), each with a description explaining what breaks if the check fails. Data comes from a single `launch_readiness_rpc` that aggregates the stats in one DB round trip.

**Alternatives considered.**
1. **Datadog / external dashboard.** Good for ops but not bootable by a non-DevOps founder during a launch.
2. **A runbook document.** Was the previous state; inconsistent execution.

**Trade-offs accepted.**
- **Pro:** single pane of glass, < 2-second refresh, immediately actionable.
- **Pro:** doubles as a business-logic contract — each failed check has a clear remediation.
- **Con:** now a surface to maintain. Mitigated by sourcing from a single RPC, so adding a check is additive.

**Status.** Accepted. Shipped April 2026.

---

## ADR-011 — Lazy-loaded routes and route-level error boundaries

**Context.** 87 page components. Loading all JS up-front would produce a 5MB+ bundle and a brutal LCP.

**Decision.** Every route in `App.tsx` is `React.lazy()`-loaded. Each route is wrapped in a `RouteErrorBoundary` so a broken page doesn't crash the shell.

**Trade-offs accepted.**
- **Pro:** initial bundle ~150kb gzipped; pages stream in on navigation.
- **Pro:** Vite code-splits automatically; no manual chunk config.
- **Con:** brief loading flash on first nav to a route. Mitigated with `<Suspense fallback>` skeletons on heavy pages.

**Status.** Accepted.

---

## ADR-012 — TypeScript with loose strict mode for MVP velocity

**Context.** Shipping speed matters more than formal verification at the current stage. Type errors are still caught; strictness escalates with maturity.

**Decision.** Keep TypeScript in strict mode but permit `any` escape hatches at integration boundaries (Supabase generated types sometimes lag the schema). Tighten per-file as modules stabilize.

**Trade-offs accepted.**
- **Pro:** we still get 80% of the safety at 20% of the ceremony.
- **Con:** a class of bugs that only full-strict catches slips through. Mitigated by Sentry and the launch-readiness checks, which catch runtime anomalies early.

**Status.** Accepted. Candidate for tightening post-launch.

---

## ADR-013 — AWS SES over Resend/SendGrid for transactional email

**Context.** Launch-time email volume is low; long-term it will grow. Vendor diversity has a cost (extra SaaS bill, extra secret to rotate).

**Decision.** Ship on AWS SES, signed from Deno via `aws4fetch` SIG V4. Reuse the same IAM credentials already used for Bedrock.

**Alternatives considered.**
1. **Resend.** Very good DX; ~5x the per-email cost at volume.
2. **SendGrid / Mailgun.** Mature but another vendor to manage.
3. **Postmark.** Excellent deliverability; pricier at scale.

**Trade-offs accepted.**
- **Pro:** $0.10 per 1000 emails.
- **Pro:** same AWS bill and IAM story as Bedrock.
- **Con:** SES production access requires a vetting step (moved out of sandbox). One-time cost.
- **Con:** we own template rendering. Mitigated by a shared [`email-styles.ts`](https://github.com/jashabalcom/swinginsitute/blob/main/supabase/functions/_shared/email-styles.ts) token set so the 80 templates are visually consistent without a WYSIWYG tool.

**Status.** Accepted. SES config marked PROTECTED.

---

## ADR-014 — pg_cron for scheduled jobs instead of a job queue

**Context.** Weekly payouts, session reminders, storage cleanup, reconciliation — all periodic. We need them to run reliably without a dedicated worker.

**Decision.** Use Postgres `pg_cron` extension. Each job is an edge function triggered on a cron schedule, with idempotent logic (`WHERE stripe_transfer_id IS NULL`) so reruns are safe.

**Alternatives considered.**
1. **SQS + Lambda.** Classic, but overkill at our volume.
2. **GitHub Actions scheduled workflow.** Works but couples ops to the CI surface; less observable.
3. **Temporal / Inngest.** Great at scale; more setup than warranted today.

**Trade-offs accepted.**
- **Pro:** zero extra infra. `pg_cron` runs in the same Postgres we already operate.
- **Pro:** last-run timestamps are a single SQL query — feeds the launch-readiness dashboard directly.
- **Con:** no retry orchestration. Mitigated by idempotent predicates and a reconciliation job that sweeps up anything a cron missed.

**Status.** Accepted. pg_cron status is surfaced in the launch-readiness dashboard.

---

## ADR-015 — Worktree-based AI-assisted development workflow

**Context.** Development uses Claude Code for pair programming. Running multiple parallel experiments (e.g., refactor the pose pipeline while also fixing a UI bug) is common. Single working directory = merge conflicts and broken state.

**Decision.** Every non-trivial task runs in an isolated git worktree under `.claude/worktrees/*`, scoped to a feature branch. Tasks merge back only when tests and lint pass.

**Trade-offs accepted.**
- **Pro:** parallel streams of work without dirtying main.
- **Pro:** fast revert — bad branch, delete the worktree.
- **Con:** disk cost (each worktree is a full checkout). Acceptable.

**Status.** Accepted. Encoded in the repo's `.claude/` configuration.

---

## ADR-016 — Ratchet-based quality gates instead of hard thresholds

**Context.** The codebase carries real type and lint debt: 167 tsc errors under `tsconfig.app.json` and 118 eslint errors by the ratchet's count. A conventional CI gate has two options and both are bad. `--max-warnings 0` is permanently red, and a permanently red gate is one the team learns to ignore — strictly worse than no gate. No gate at all lets the debt compound silently.

**Decision.** Gate on the *derivative*, not the *value*. `scripts/tsc-ratchet.mjs` and `scripts/eslint-ratchet.mjs` compare the current error count against a committed baseline (`.tsc-baseline`, `.eslint-baseline`) and fail only if it increased. The baseline can only move down; when a change lowers it, the ratchet rewrites the file and it lands in the diff.

**Alternatives considered.**
1. **Hard zero-error gate.** Requires paying off all 285 errors before any gate exists. Months of yak-shaving before the first signal.
2. **Warning-only CI.** No enforcement; debt grows.
3. **Per-directory strictness escalation.** Good long-term shape, much more config, and doesn't solve the "what about today" problem.

**Trade-offs accepted.**
- **Pro:** always green on a clean change, always red on a regression. That is precisely the signal a gate should carry, from day one, with existing debt intact.
- **Pro:** improvement is visible and irreversible — a lowered baseline is a committed file, so the ratchet cannot silently slip back.
- **Con:** the existing 285 errors are not fixed, just fenced. Honest framing: this bounds the debt, it does not repay it.
- **Con:** the number is only meaningful against a pinned dependency tree — see ADR-017, which exists entirely because of a way this can produce a false reading.

**Status.** Accepted. Both ratchets are blocking on PRs and pushes.

---

## ADR-017 — Node gates before Deno gates in CI, enforced by a fail-closed check

**Context.** `deno check --node-modules-dir=auto` resolves the edge functions' `npm:@supabase/supabase-js@2` specifiers into the **shared** `node_modules` directory. It re-resolves independently of `package-lock.json` and installs the newest match for a caret range. Observed on 2026-08-15: it moved `@supabase/supabase-js` 2.90.1 → 2.111.0 and `typescript` 5.8.3 → 5.9.3 in a tree `npm ci` had just pinned.

The two Supabase versions expose different `SupabaseClient` generics. A `tsc` run after a `deno check` therefore reports errors in `src/` files the changeset never touched: `tsc-ratchet` went **169 → 182**, with all 13 "new" errors in untouched frontend code, attributed to a diff that added nothing.

**Decision.** Every Node/tsc/eslint/vitest gate runs *before* any Deno step in the quality job. The ordering is commented as load-bearing in `.github/workflows/deploy.yml`, and `scripts/tsc-ratchet.mjs` fails closed when it detects the deno-shaped tree, naming `npm ci` as the fix.

**Alternatives considered.**
1. **Drop `--node-modules-dir=auto`.** Deno then can't resolve the npm specifiers the edge functions genuinely use.
2. **Separate CI jobs with separate checkouts.** Clean isolation, but doubles checkout + install time on every PR for a problem that ordering solves for free.
3. **Run `npm ci` between the Deno steps and any later Node step.** Valid, and documented as the escape hatch if the order ever must change. Costs an extra install.
4. **Comment only.** Comments get skipped. This failure costs an afternoon precisely because it is *plausible* — it looks exactly like a real regression.

**Trade-offs accepted.**
- **Pro:** zero runtime cost; it is an ordering constraint, not extra work.
- **Pro:** the fail-closed check means the trap is caught even if someone reorders the workflow and misses the comment.
- **Con:** an implicit coupling between two toolchains that a reader must be told about. Mitigated by the comment, the check, and this ADR.

**Reproduced deterministically:** `npm ci` → 169, `deno check` → 182, `npm ci` → 169.

**Also affects iOS.** A `npx cap sync ios` run against a deno-shaped `node_modules` rewrites `ios/App/CapApp-SPM/Package.swift` to `node_modules/.deno/@capacitor+app@8.1.1/…` paths that exist on no other machine. Never commit that file after a Deno run.

**Status.** Accepted. This is the ADR to point at when asked "tell me about a hard bug."

---

## ADR-018 — Grant-level lockdown as a distinct security layer above RLS

**Context.** The 2026-08-15 audit found that a small number of `SECURITY DEFINER` functions were reachable by the anonymous role. The project's guard pattern intentionally lets NULL-uid callers through so cron and service-role can use it, and is safe **only** when the anonymous role is revoked at the grant level. Most functions had that revoke; a few added in later migrations granted execute to `authenticated` and never revoked the default — and because new public-schema functions are auto-granted to the anonymous role, granting to `authenticated` restricted nothing. `SECURITY DEFINER` bypasses RLS, so the correct table policies were irrelevant. *(Specifics are held in the private repo.)*

**Decision.** Treat `GRANT`/`REVOKE` as a first-class security layer sitting *above* RLS, not as deployment trivia. Every privileged RPC is explicitly revoked from `PUBLIC` **and** `anon`. Verification uses `has_function_privilege` / `has_column_privilege` after apply — never the migration's own success return.

**Alternatives considered.**
1. **Harden each guard to fail closed on NULL uid.** Correct and worth doing, but it fixes the ten functions that exist rather than the default that will mis-grant the eleventh.
2. **Rely on RLS.** Structurally cannot work: `SECURITY DEFINER` bypasses it. This is the trap.
3. **Move all privileged logic into edge functions.** Loses the atomicity that makes the credit RPCs race-free (ADR-005).

**Trade-offs accepted.**
- **Pro:** closes the class, not the instance. A new RPC is now presumed exposed until proven revoked.
- **Pro:** blast radius of the revoke was verifiably zero — the only callers are service-role edge functions.
- **Con:** a `REVOKE` that must accompany every new `SECURITY DEFINER` function is a discipline dependency. Mitigated by the `CLAUDE.md` pre-flight checks and post-apply privilege assertions.

**The generalizable lesson:** a column-level `REVOKE` is a **silent no-op** while a table-level `GRANT` stands — it returns success and changes nothing. Same for functions while `PUBLIC` holds the grant. Always revoke broad, then re-grant narrow, then verify.

**Status.** Accepted. Encoded as pre-flight check #1 in `CLAUDE.md`.

---

## ADR-019 — Two-stage rollout for permission-narrowing migrations

**Context.** Any authenticated user could `SELECT` every column of every `profiles` row — including children's legal names, ages, and parent email/phone. The obvious fix (revoke the broad grant) breaks login for every existing session, because 54 places in the frontend read profile fields cross-user.

**Decision.** Split into two independently shippable stages with a hard ordering dependency:
- **Stage A** — introduce a narrowed `member_directory` view and migrate all 54 read sites onto it. Ship the frontend. No permission changes; nothing can break.
- **Stage B** — only after the narrowed frontend is live in production, revoke the broad `profiles` select grant.

**Alternatives considered.**
1. **Single migration.** Instant lockout for anyone on a cached bundle. Unacceptable.
2. **Revoke first, patch the fallout.** Inverts the risk onto users.
3. **Column-level revoke alone.** A no-op while the table-level grant stands (ADR-018).

**Trade-offs accepted.**
- **Pro:** at no point does a deployed client lack a permission it needs. The window between stages is safe in both directions.
- **Pro:** Stage A is reviewable as a pure refactor; Stage B is a two-line permission change. Both are far easier to reason about than their union.
- **Con:** the vulnerability stays open across the gap between stages. Accepted knowingly: pre-launch, with ~0 real members, the exposure window carried near-zero real risk while a botched single-shot fix would have been an outage.
- **Con:** ordering is a human dependency. Mitigated by stating it in `CLAUDE.md` as the one hard dependency in the audit fix plan.

**Status.** Accepted, both stages shipped (2026-08-21 / 2026-08-22). Stage B carries a test written to fail if the hole reopens.

---

## ADR-020 — Adversarial verification for audit findings

**Context.** A broad automated audit produces findings at a rate no solo operator can triage. Worse, it produces *confident* findings — and acting on a false positive costs the same engineering hours as a real one while adding risk.

**Decision.** Every finding goes to a second reviewer whose only job is to **refute** it. A finding is "confirmed" only after surviving that. The 2026-08-15 audit raised 89 and confirmed 36 — **14 were actively refuted**, the rest unresolved or duplicated.

**Alternatives considered.**
1. **Triage by severity label.** Severity is asserted by the same process that raised the finding; it inherits the same error rate.
2. **Fix everything raised.** 89 changes, most unnecessary, each one risk.
3. **Manual spot-check.** What this replaced; inconsistent under time pressure.

**Trade-offs accepted.**
- **Pro:** a **16% refutation rate** is the number that matters. Without this step, roughly one in seven fixes would have been churn against a non-problem.
- **Pro:** verification output is evidence — the blocker finding carries an end-to-end reproduction, deliberately parameterized so it proves the guard failed while touching zero rows.
- **Pro:** the verifier caught an error *in the remediation*: the finding proposed `REVOKE ... FROM anon`, which would not have closed the hole, because both functions also carried the `PUBLIC` grant.
- **Con:** roughly doubles audit cost.
- **Con:** two agents died on an expired OAuth token, so 5 funnel findings were never verified. They are labeled *raised*, not *confirmed*, in the audit doc. Stating the gap is part of the method.

**Status.** Accepted. This is the reason to trust the audit's numbers.

---

## ADR-021 — COPPA compliance as an architectural constraint, not a feature

**Context.** The platform trains youth athletes. Minors are a core user population, not an edge case, which puts COPPA and App Store Guideline 1.2/2.1 in the critical path.

**Decision.** Push child-safety into structural layers that cannot be bypassed by a UI path:
- `swing-videos` bucket is `public=false` with storage RLS scoped to owner / parent / assigned coach / admin; reads via signed URLs.
- Age gate before account creation, with parent-verified child account creation and out-of-band consent email.
- Ad and analytics pixels suppressed on under-13 sessions, on every route.
- Sentry does not record minors at all.
- `process-account-deletions` carries a **coverage ratchet** — a migration-enforced check that fails if a new table holding user data escapes the deletion path.

**Alternatives considered.**
1. **Age gate in the UI only.** Bypassable, and the audit found exactly this: a 10-year-old who closed the parent-email step retained full access.
2. **Public bucket + unguessable URLs.** Security by obscurity over minors' video. Not defensible to a reviewer or a parent.
3. **Adults-only ToS.** Contradicts the actual product.

**Trade-offs accepted.**
- **Pro:** the deletion coverage ratchet is the one that keeps paying — it converts "remember to update the deletion path" into a build failure.
- **Pro:** signed URLs cost a round trip but make sharing revocable.
- **Con:** signed-URL expiry is a real UX failure mode. It bit us: community media stored as absolute URLs bypassed the signing path and broke.
- **Con:** suppressing pixels for minors means marketing attribution is structurally incomplete for part of the audience. Correct trade, and worth saying out loud rather than discovering it in a funnel review.

**Status.** Accepted. The audit's compliance dimension raised 11 findings and confirmed 6 — the highest confirm rate of any dimension.

---

## ADR-022 — Sweepers over queues for stalled business processes

**Context.** ~15 multi-step processes can stall: a booking request nobody accepts, a dispute past SLA, an expiring credit, a coach who abandoned Connect onboarding, a session never marked complete. The product goal is explicit — the operator should run on alerts, not manual queues.

**Decision.** Each stalled-state process gets a `pg_cron`-driven sweeper edge function that finds stalled rows by predicate and advances or expires them. Idempotent by construction (`WHERE stripe_transfer_id IS NULL`, `WHERE expires_at <= now()`), so every run is safe to repeat.

**Alternatives considered.**
1. **Event-driven state machine (Temporal, Inngest).** Right answer at scale; more infrastructure than a pre-launch solo product can justify operating.
2. **SQS + Lambda per process.** ~15 queues, ~15 DLQs, ~15 alarms to operate.
3. **Manual admin queues.** Directly contradicts the "zero operator involvement" goal.

**Trade-offs accepted.**
- **Pro:** no extra infra — `pg_cron` runs in the Postgres we already operate, and last-run timestamps are one SQL query, feeding the launch-readiness dashboard.
- **Pro:** a missed run is harmless. The next run picks up everything, because the predicate describes the *state*, not the *event*.
- **Con:** latency is bounded by cron interval, not event arrival. Fine for hour-scale business processes; would not be for anything user-facing.
- **Con:** ~15 sweepers is real surface area. Mitigated by a shared shape and by the canary probing two of them directly.
- **Con:** no retry orchestration or backoff. The idempotent predicate is the retry story.

**Status.** Accepted. Sweeper health is surfaced on the launch-readiness dashboard and probed by `canary-runner`.

---

## Superseded / amended

**ADR-012 (TypeScript loose strict mode)** — still accepted, but the "we'll tighten it post-launch" plan now has teeth it lacked when written. The debt is measured (167 tsc / 118 eslint) and fenced by ratchets (ADR-016) rather than tracked by intention. Tightening lowers a committed number; drifting fails CI. That is the difference between a plan and a mechanism.

---

## Revisit cadence

These ADRs get re-read at each major phase gate (pre-launch, post-first-100 MAU, pre-scale migration). An ADR is never "closed forever" — it's "accepted until a better answer shows up."
