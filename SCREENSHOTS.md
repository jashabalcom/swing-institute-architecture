# Portfolio Screenshot Shot List

> Exactly what to capture and where to put it, to make the GitHub README + portfolio site show the depth of the platform at a glance. A hiring manager should get it in under 30 seconds of scrolling.

## How to use this list

1. Capture each shot at the exact viewport listed — consistency reads as craft.
2. Save all shots into `.screenshots/` (create the folder).
3. Name files exactly as specified — the README will reference them by filename.
4. Use the "why this shot" column when writing captions on LinkedIn / portfolio site.

## Capture settings (do this once)

- **Desktop screenshots:** Chrome at **1440×900**, "Device Toolbar off." Use the browser's clean incognito window so no bookmarks bar shows.
- **iPhone screenshots:** iPhone 15/16 Pro simulator at native resolution (use the Xcode simulator's ⌘S to save). Use dark status bar with full signal/battery.
- **Admin screenshots:** log in as admin, use a dataset with at least a week of real-looking activity (not zero state).
- **Redact PII:** blur member names/emails if present. Use the Swing Institute navy (#1B2A4A) solid blur.
- **File format:** PNG, not JPG. Portfolio sites compress further; starting lossy is sloppy.

---

## Tier 1 — The "first impression" shots (README hero grid)

Put these in the README as a 2×3 grid up top. They are the ones a hiring manager will actually see.

| # | Filename | Surface | Route / State | Viewport | Why this shot |
|---|---|---|---|---|---|
| 1 | `01-dashboard-hero.png` | Web | `/dashboard` — logged in, with a recent AI score, streak, smart drills | 1440×900 | The product's center of gravity in one frame. Shows the AI score ring + progress chart + smart drills all at once. |
| 2 | `02-swing-analysis-web.png` | Web | `/swing-vault/[clip]` — AI analysis expanded, showing category scores + biomechanics | 1440×900 | The hero feature. Shows you built a working AWS Bedrock vision pipeline. |
| 3 | `03-swing-analysis-ios.png` | iOS | Same clip, iOS app, analysis result | iPhone 16 Pro | Proof of cross-platform. One screenshot sells the "one codebase, three surfaces" story. |
| 4 | `04-coach-review-queue.png` | Web | `/coach-dashboard` with ReviewQueue open, urgency filter applied | 1440×900 | Shows coach-side tooling and that you built for the supply side too. |
| 5 | `05-admin-launch-readiness.png` | Web | `/admin/launch-readiness` — some checks green, one warn, one fail | 1440×900 | Senior signal. Few solo founders build an ops dashboard like this. |
| 6 | `06-pricing-v2.png` | Web | `/pricing` — V2, with segmented toggle in "Online" mode | 1440×900 | Conversion craft + design polish. |

---

## Tier 2 — Architecture & system shots

For the public architecture repo and `.ARCHITECTURE.md`. You don't need to capture these if Mermaid renders — but if you want fallback images:

| # | Filename | What | How to capture |
|---|---|---|---|
| A1 | `arch-01-logical.png` | Logical architecture diagram | Open `ARCHITECTURE.md` on GitHub, screenshot the first Mermaid flowchart |
| A2 | `arch-02-aws-physical.png` | Physical AWS architecture | Same doc, second diagram (the `architecture-beta` one with AWS icons) |
| A3 | `arch-03-ai-sequence.png` | Swing analysis sequence | Same doc, the sequenceDiagram for AI flow |
| A4 | `arch-04-payout-sequence.png` | Coach payout sequence | Same doc, the sequenceDiagram for Stripe |
| A5 | `arch-05-notification-fanout.png` | 3-channel notification flow | Same doc, the fan-out flowchart |
| A6 | `arch-06-erd.png` | ER diagram | Same doc, the erDiagram |

If you want pro-grade architecture art instead of Mermaid screenshots, use the official AWS Architecture Icons (PNG pack from <https://aws.amazon.com/architecture/icons/>) in Figma, Excalidraw, or Diagrams.net (`draw.io`). Target 1920×1080 exports.

---

## Tier 3 — Product depth shots (for portfolio site, case study, or second scroll of README)

Group these under headings so they tell a story.

### Athlete experience
| # | Filename | Route | Viewport |
|---|---|---|---|
| B1 | `10-home-landing.png` | `/` (top of page, hero + app CTA visible) | 1440×900 |
| B2 | `11-swing-recorder.png` | Swing recorder mid-capture (countdown state) | iPhone 16 Pro |
| B3 | `12-swing-vault-grid.png` | `/swing-vault` with filters + multiple clips | 1440×900 |
| B4 | `13-dashboard-ios.png` | `/dashboard` on iOS — same content, mobile layout | iPhone 16 Pro |
| B5 | `14-academy-levels.png` | `/academy` — levels grid with progress rings | 1440×900 |
| B6 | `15-academy-lesson.png` | `/academy/level/.../module/.../lesson/...` — video + progress | 1440×900 |
| B7 | `16-community-feed.png` | `/community` — post feed with a pinned post and one with media | 1440×900 |
| B8 | `17-community-dm.png` | Direct message panel open with an active conversation | 1440×900 |
| B9 | `18-book-calendar.png` | `/book` — premium calendar with availability dots | 1440×900 |

### Coach experience
| # | Filename | Route | Viewport |
|---|---|---|---|
| C1 | `20-coach-dashboard.png` | `/coach-dashboard` — earnings + review queue side by side | 1440×900 |
| C2 | `21-coach-review-detail.png` | CoachReviewInterface on a clip with AIPreAnalysis card visible | 1440×900 |
| C3 | `22-coach-availability.png` | `/admin/coach-schedule` or AvailabilityEditor | 1440×900 |
| C4 | `23-stripe-connect-setup.png` | StripeConnectSetup flow mid-step | 1440×900 |

### Admin / ops experience
| # | Filename | Route | Viewport |
|---|---|---|---|
| D1 | `30-admin-dashboard.png` | `/admin` top-level | 1440×900 |
| D2 | `31-admin-revenue.png` | `/admin/revenue` with charts | 1440×900 |
| D3 | `32-admin-payroll.png` | `/admin/payroll` showing a week of payouts with Stripe Transfer IDs | 1440×900 |
| D4 | `33-admin-moderation.png` | `/admin/moderation` queue | 1440×900 |
| D5 | `34-admin-partners.png` | `/admin/partner-applications` — cross-program queue | 1440×900 |
| D6 | `35-admin-members.png` | `/admin/members` with row dialog open | 1440×900 |
| D7 | `36-admin-launch-readiness-full.png` | Full page scroll of launch readiness | 1440×2200 (long shot) |

### Partner program landing pages (marketing craft)
| # | Filename | Route | Viewport |
|---|---|---|---|
| E1 | `40-coach-landing.png` | `/become-a-coach` — hero + income calculator visible | 1440×900 |
| E2 | `41-nil-landing.png` | `/nil` | 1440×900 |
| E3 | `42-ambassador-landing.png` | `/ambassadors` | 1440×900 |
| E4 | `43-affiliate-landing.png` | `/affiliate` | 1440×900 |
| E5 | `44-application-form.png` | Multi-step form mid-wizard (step 2 of 3) | 1440×900 |

### iOS app experience
| # | Filename | Screen | Viewport |
|---|---|---|---|
| F1 | `50-ios-intro-carousel.png` | NativeIntro carousel card 1 | iPhone 16 Pro |
| F2 | `51-ios-welcome.png` | NativeWelcome screen | iPhone 16 Pro |
| F3 | `52-ios-dashboard.png` | Dashboard with tab bar | iPhone 16 Pro |
| F4 | `53-ios-swing-recorder.png` | Live capture with pose overlay | iPhone 16 Pro |
| F5 | `54-ios-analysis-result.png` | Analysis result with category cards | iPhone 16 Pro |
| F6 | `55-ios-community.png` | Community feed on iOS | iPhone 16 Pro |
| F7 | `56-ios-push-notification.png` | Lock-screen notification "Coach left feedback on your swing" | iPhone 16 Pro |

---

## Tier 4 — Infra & ops shots (the senior-signal shots)

These are what separate portfolios — proof you operated the system, not just built it.

| # | Filename | What to capture | Source |
|---|---|---|---|
| G1 | `60-aws-bedrock-console.png` | Bedrock invocation metrics or model access page for Sonnet 4 | AWS Console → Bedrock → Model access |
| G2 | `61-aws-ses-verified-domains.png` | SES verified sending domain + suppression dashboard | AWS Console → SES |
| G3 | `62-aws-amplify-deployment.png` | Amplify deployment history showing CI/CD runs | AWS Console → Amplify |
| G4 | `63-aws-cloudfront-metrics.png` | CloudFront request/bandwidth dashboard | AWS Console → CloudFront |
| G5 | `64-supabase-project.png` | Supabase project overview (tables count, DB size) | Supabase Dashboard |
| G6 | `65-supabase-functions.png` | Edge functions list showing all 35 | Supabase Dashboard → Edge Functions |
| G7 | `66-supabase-rls.png` | Table policies tab showing RLS on a user-scoped table | Supabase Dashboard → Table Editor → Policies |
| G8 | `67-stripe-connect-dashboard.png` | Stripe Connect dashboard showing connected accounts | Stripe Dashboard → Connect |
| G9 | `68-sentry-dashboard.png` | Sentry project with some resolved errors | Sentry Dashboard |
| G10 | `69-github-actions.png` | GitHub Actions runs showing passing builds | <https://github.com/jashabalcom/swinginsitute/actions> |

Redact account IDs, API keys, and any customer PII in every AWS/Supabase/Stripe screenshot.

---

## Tier 5 — The "did you actually ship it" shots

| # | Filename | What | How |
|---|---|---|---|
| H1 | `70-app-store-listing.png` | App Store Connect listing | App Store Connect → My Apps |
| H2 | `71-testflight-build.png` | TestFlight build history | App Store Connect → TestFlight |
| H3 | `72-live-site-production.png` | Real `www.swinginstitutebaseball.com` with browser chrome visible | Any browser |

---

## Once you have the shots — where they go

### README (main repo + public architecture repo)
At the top, under the shields, add a hero grid of Tier 1 (six shots):
```markdown
<p align="center">
  <img src=".screenshots/01-dashboard-hero.png" width="32%" />
  <img src=".screenshots/02-swing-analysis-web.png" width="32%" />
  <img src=".screenshots/05-admin-launch-readiness.png" width="32%" />
</p>
<p align="center">
  <img src=".screenshots/03-swing-analysis-ios.png" width="24%" />
  <img src=".screenshots/04-coach-review-queue.png" width="32%" />
  <img src=".screenshots/06-pricing-v2.png" width="32%" />
</p>
```

### LinkedIn post (post-capture)
Use Tier 1 + one Tier 4 infra shot (G5 or G6). Two-column image collage.

### Portfolio site / case study
Full Tier 3 grouped by "Athlete / Coach / Admin / iOS." Tier 4 under "Operations." Tier 5 at the top as proof-of-shipping.

### Interviews
Share a deck (Keynote/Google Slides) with Tier 1 + Tier 2 + Tier 4. Keep it to ~12 slides; depth is in the repo.

---

## Quick capture checklist

- [ ] Tier 1 hero shots (6) — do these first, they drive everything else
- [ ] Tier 2 architecture diagrams (6) — Mermaid renders on GitHub; screenshot only if you want them as raster images
- [ ] Tier 3 athlete experience (9)
- [ ] Tier 3 coach experience (4)
- [ ] Tier 3 admin experience (7)
- [ ] Tier 3 partner landings (5)
- [ ] Tier 3 iOS experience (7)
- [ ] Tier 4 infra/ops (10)
- [ ] Tier 5 shipping proof (3)

**Total: ~57 shots.** Budget 3–4 hours to capture cleanly.

## One more thing

When you caption these on LinkedIn or the portfolio site, **describe the system, not the screenshot**. "Dashboard on launch day" is lazy. "Dashboard: AI-score ring is generated from AWS Bedrock vision analysis on 12 key frames, persisted as JSONB, surfaced via a single rolling-average query" is the way.
