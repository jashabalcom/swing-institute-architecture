# Swing Institute — AWS Deployment Plan

## Current Stack
- **Frontend**: React 18 + Vite (static SPA)
- **Backend**: Supabase (PostgreSQL, Auth, Edge Functions, Realtime, Storage)
- **Payments**: Stripe
- **CRM**: GoHighLevel

## Phase 1: Deploy Frontend to AWS (Immediate)

### AWS Amplify Hosting
The simplest path to get the app on AWS with SSL, CDN, and CI/CD.

**Setup:**
1. Connect GitHub repo to AWS Amplify
2. Build settings: `npm run build`, output: `dist`
3. Environment variables:
   - `VITE_SUPABASE_URL`
   - `VITE_SUPABASE_PUBLISHABLE_KEY`
4. Custom domain: `app.swinginstitute.com` (or your domain)
5. Enable branch previews for staging

**Cost: ~$0-15/month** (free tier covers most traffic)

**AWS Services:**
- AWS Amplify (hosting + CI/CD)
- Amazon CloudFront (CDN, included with Amplify)
- AWS Certificate Manager (SSL, free)
- Amazon Route 53 (DNS, ~$0.50/month per hosted zone)

---

## Phase 2: Migrate Email to AWS SES (When Ready)

### Replace Resend with Amazon SES
- Production email at ~$0.10 per 1,000 emails (vs Resend's limits)
- Update edge functions to use SES SDK instead of Resend

**Setup:**
1. Verify sending domain in SES
2. Request production access (out of sandbox)
3. Create SMTP credentials or use SES SDK
4. Update Supabase Edge Functions to send via SES

**Cost: ~$1-5/month** for typical SaaS email volume

---

## Phase 3: Full AWS Migration (Future — When Outgrowing Supabase)

This phase migrates the entire backend from Supabase to AWS-native services. Only do this when you need:
- Custom server-side logic beyond edge functions
- More control over database scaling
- Compliance requirements (data residency, SOC 2)
- Cost optimization at scale (1000+ users)

### Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Amazon CloudFront                  │
│                    (CDN + SSL)                        │
├─────────────────────┬───────────────────────────────┤
│   S3 (Static SPA)   │   API Gateway (REST/WS)       │
│   React App          │   ├── /auth → Cognito         │
│                      │   ├── /api → Lambda           │
│                      │   └── /ws → WebSocket API     │
├──────────────────────┴───────────────────────────────┤
│                    AWS Lambda                         │
│   ├── booking-handler                                │
│   ├── stripe-webhook                                 │
│   ├── email-sender (SES)                             │
│   ├── ghl-sync                                       │
│   └── ai-advisor (Bedrock) [future]                  │
├──────────────────────────────────────────────────────┤
│                    Data Layer                         │
│   ├── Aurora PostgreSQL Serverless v2 (primary DB)   │
│   ├── ElastiCache Redis (sessions, caching)          │
│   ├── S3 (video storage, documents)                  │
│   └── DynamoDB (real-time events, notifications)     │
├──────────────────────────────────────────────────────┤
│                    Auth & Security                    │
│   ├── Amazon Cognito (user pools, MFA)               │
│   ├── AWS WAF (rate limiting, protection)            │
│   ├── AWS KMS (encryption at rest)                   │
│   ├── AWS Secrets Manager (API keys)                 │
│   └── CloudTrail (audit logging)                     │
├──────────────────────────────────────────────────────┤
│                    Observability                      │
│   ├── CloudWatch (metrics, logs, alarms)             │
│   ├── X-Ray (distributed tracing)                    │
│   └── CloudWatch Synthetics (uptime monitoring)      │
└──────────────────────────────────────────────────────┘
```

### Migration Order
1. **Frontend** → S3 + CloudFront (or keep Amplify)
2. **Auth** → Amazon Cognito (replace Supabase Auth)
3. **Database** → Aurora PostgreSQL Serverless v2 (migrate with DMS)
4. **Edge Functions** → AWS Lambda + API Gateway
5. **Storage** → S3 (migrate from Supabase Storage)
6. **Realtime** → API Gateway WebSocket + DynamoDB Streams

### IaC
All infrastructure defined in **AWS CDK (TypeScript)** for reproducible deployments.

### Cost Estimate (Phase 3, 100-500 users)
| Service | Monthly Cost |
|---------|-------------|
| Amplify or CloudFront+S3 | $5-15 |
| Aurora Serverless v2 | $30-80 |
| Lambda | $1-5 |
| API Gateway | $3-10 |
| Cognito | Free (50k MAU) |
| SES | $1-5 |
| ElastiCache | $15-30 |
| CloudWatch | $5-10 |
| **Total** | **$60-155/month** |

vs. current Supabase Pro: **$25/month**

**Recommendation**: Stay on Supabase until you hit 500+ active users or need features Supabase can't provide. Then migrate incrementally.

---

## Phase 4: AI Enhancement (Future — Post-Migration)

### Amazon Bedrock Integration
- AI-powered swing analysis feedback
- Natural language coaching assistant
- Automated drill recommendations based on progress

### Amazon Rekognition
- Automatic swing video tagging
- Form analysis from uploaded videos

---

## Certification Relevance

| AWS Service | SAA-C03 Domain | What This Demonstrates |
|-------------|---------------|----------------------|
| Amplify + CloudFront | Design High-Performing | Static site with CDN |
| Aurora Serverless v2 | Design Cost-Optimized | Auto-scaling database |
| Lambda + API Gateway | Design Resilient | Serverless compute |
| Cognito | Design Secure | Managed auth with MFA |
| SES | Design Cost-Optimized | Managed email at scale |
| CDK | Operational Excellence | Infrastructure as Code |
| WAF + KMS | Design Secure | Defense in depth |

---

## Immediate Next Steps
1. Deploy to AWS Amplify (Phase 1) — can do today
2. Set up custom domain + SSL
3. Configure Amplify environment variables
4. Enable branch previews for staging workflow
