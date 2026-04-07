# MergePDFly

> **Merge, split, compress, and secure PDFs in your browser — no uploads, no privacy risks, no subscription required for basic use.**

Full production SaaS platform built with Next.js 16 (App Router), TypeScript, Tailwind CSS, Supabase, Stripe, and Cloudflare R2.

---

## Architecture

```
Browser (pdf-lib WASM)        →  Free-tier PDF processing (never leaves device)
Next.js API Routes (Vercel)   →  Auth, billing, rate limiting, batch jobs
Supabase (Postgres + Auth)    →  User data, file records, subscriptions
Stripe                        →  Payments, webhooks, customer portal
Upstash Redis                 →  Sliding-window rate limiting per user/tier
Cloudflare R2                 →  Temp storage for cloud-processed files
Cloud Run (Python workers)    →  Ghostscript + qpdf + Tesseract (paid tiers)
```

---

## Route Map (40 routes, 0 build errors)

| Route | Type | Description |
|-------|------|-------------|
| `/` | Static | Landing page — hero, features, pricing, trust bar |
| `/merge-pdf` | Static | Client-side PDF merge tool |
| `/split-pdf` | Static | Client-side PDF splitter |
| `/compress-pdf` | Static | Cloud-powered compressor |
| `/protect-pdf` | Static | Client-side AES-256 encryption + watermark |
| `/ocr-pdf` | Static | Pro-tier OCR (Tesseract) upgrade prompt |
| `/pricing` | Static | Plans comparison table |
| `/api-docs` | Static | REST API documentation + code samples |
| `/enterprise` | Static | Enterprise sales page |
| `/security` | Static | Zero-knowledge architecture explainer |
| `/blog` | Static | Content hub index |
| `/blog/[slug]` | SSG | 4 pre-rendered long-form posts |
| `/login` | Static | Magic link + Google + GitHub OAuth |
| `/signup` | Static | Account creation with plan context |
| `/dashboard` | Dynamic | Usage stats + recent files + quick actions |
| `/dashboard/files` | Dynamic | Cloud file list with download/delete |
| `/dashboard/batch` | Dynamic | Multi-file batch job submission |
| `/dashboard/api-keys` | Dynamic | Generate / revoke API keys |
| `/dashboard/settings` | Dynamic | Profile + Stripe billing portal |
| `/dashboard/help` | Dynamic | Knowledge base + support |
| `/admin` | Dynamic | Redirects → /admin/users |
| `/admin/users` | Dynamic | User search + tier/status view |
| `/admin/subscriptions` | Dynamic | Stripe subscription audit table |
| `/admin/analytics` | Dynamic | 30-day MAU, usage, tier breakdown |
| `/admin/security` | Dynamic | IP blocklist + rate limit config |
| `/api/v1/process/batch` | API | Batch job submission (Pro+) |
| `/api/v1/jobs/[jobId]` | API | Job status polling |
| `/api/v1/compress` | API | Single-file cloud compression |
| `/api/v1/keys` | API | List / create API keys |
| `/api/v1/keys/[keyId]` | API | Revoke API key |
| `/api/billing/create-checkout` | API | Stripe Checkout session |
| `/api/billing/portal` | API | Stripe Customer Portal redirect |
| `/api/webhooks/stripe` | API | Subscription lifecycle sync |
| `/api/auth/callback` | API | Supabase OAuth callback |
| `/sitemap.xml` | Static | SEO sitemap (all public routes + blog) |
| `/robots.txt` | Static | Crawler directives |

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Framework | Next.js 16 (App Router), TypeScript |
| Styling | Tailwind CSS v4 |
| PDF engine | pdf-lib (client-side, zero server contact) |
| Auth | Supabase Auth (Google, GitHub, magic link) |
| Database | Supabase Postgres 15 with Row-Level Security |
| Payments | Stripe Checkout + Customer Portal |
| Rate limiting | Upstash Redis sliding-window |
| File storage | Cloudflare R2 (S3-compatible, zero egress) |
| PDF workers | Python + FastAPI + Ghostscript + Tesseract on Cloud Run |
| Hosting | Vercel (frontend) + Google Cloud Run (workers) |
| CI/CD | GitHub Actions |

---

## Prerequisites

- Node.js 22+
- npm 10+
- Accounts: Supabase, Stripe, Upstash, Cloudflare (all have free tiers)

---

## 1 — Run Locally

```bash
# Clone the repo
git clone <your-repo-url>
cd mergepdfly

# Install dependencies
npm install

# Create local environment file
# (copy env-config.md values into a new file called .env.local)
copy env-config.md .env.local   # then edit with real values

# Start dev server
npm run dev
```

Open **http://localhost:3000**

> The free-tier PDF tools (merge, split, protect, watermark) work **immediately without any backend** — they run entirely in the browser using WebAssembly. You only need Supabase/Stripe configured for auth and billing.

---

## 2 — Required Service Setup

### Supabase (Auth + Database)

1. Create a project at [app.supabase.com](https://app.supabase.com)
2. **SQL Editor** → paste `src/lib/db/schema.sql` → **Run**  
   This creates all tables, indexes, RLS policies, and the auto-profile trigger.
3. **Settings → API** → copy:
   - Project URL → `NEXT_PUBLIC_SUPABASE_URL`
   - `anon` public key → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `service_role` secret key → `SUPABASE_SERVICE_ROLE_KEY`
4. **Authentication → Providers** → enable **Google** and **GitHub**  
   (requires OAuth app credentials from Google Cloud Console / GitHub Developer Settings)
5. **Authentication → URL Configuration** → add redirect URLs:
   ```
   http://localhost:3000/api/auth/callback
   https://yourdomain.com/api/auth/callback
   ```

### Stripe (Payments)

1. Create account at [stripe.com](https://stripe.com)
2. **Developers → API keys** → copy both keys:
   - Publishable → `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`
   - Secret → `STRIPE_SECRET_KEY`
3. **Products** → create three products:

   | Product | Price | Billing | Lookup key |
   |---------|-------|---------|------------|
   | Pro Monthly | $9.00 | Monthly | `pro_monthly` |
   | Pro Yearly | $79.00 | Yearly | `pro_yearly` |
   | Team Monthly | $49.00 | Monthly | `team_monthly` |

   Copy each **Price ID** into the corresponding env var.

4. **Developers → Webhooks** → Add endpoint:
   - URL: `https://yourdomain.com/api/webhooks/stripe`
   - Events: `checkout.session.completed`, `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`
   - Copy **Signing secret** → `STRIPE_WEBHOOK_SECRET`

5. For local webhook testing:
   ```bash
   stripe listen --forward-to localhost:3000/api/webhooks/stripe
   ```

### Upstash Redis (Rate Limiting)

1. Create database at [console.upstash.com](https://console.upstash.com)
2. Select **Global** region for lowest latency
3. Copy **REST URL** → `UPSTASH_REDIS_REST_URL`
4. Copy **REST Token** → `UPSTASH_REDIS_REST_TOKEN`

### Cloudflare R2 (File Storage)

1. [Cloudflare Dashboard](https://dash.cloudflare.com) → **R2** → **Create bucket**
2. Name it `mergepdfly-files`
3. **R2 → Manage R2 API Tokens** → create token with `Object Read & Write`
4. Copy Account ID, Access Key, and Secret Key into env vars
5. Optionally connect `files.yourdomain.com` as a custom domain

---

## 3 — Deploy to Vercel

### Option A: Vercel CLI

```bash
npm i -g vercel
vercel --prod
```

Add all environment variables when prompted (or in Vercel project dashboard under **Settings → Environment Variables**).

### Option B: GitHub Actions (automatic on push)

The CI/CD pipeline is at `.github/workflows/deploy.yml`.

Add these **GitHub Secrets**:

```
VERCEL_TOKEN          # from vercel.com/account/tokens
VERCEL_ORG_ID         # from .vercel/project.json after first deploy
VERCEL_PROJECT_ID     # from .vercel/project.json after first deploy
GCP_SA_KEY            # Google Cloud service account JSON (for Cloud Run)
```

Plus all environment variables from `env-config.md`.

Every push to `main` → tests → Vercel deploy → Cloud Run workers deploy.

---

## 4 — Deploy Workers (Cloud Run)

The Python workers in `./workers/` handle Ghostscript compression, qpdf merging, and Tesseract OCR for paid-tier cloud jobs.

```bash
gcloud run deploy pdf-workers \
  --source ./workers \
  --region us-central1 \
  --memory 2Gi \
  --cpu 2 \
  --min-instances 0 \
  --max-instances 50 \
  --set-env-vars "R2_ACCOUNT_ID=...,R2_ACCESS_KEY_ID=...,R2_SECRET_ACCESS_KEY=...,R2_BUCKET_NAME=mergepdfly-files,R2_PUBLIC_URL=https://files.yourdomain.com"
```

---

## Folder Structure

```
src/
├── app/
│   ├── (auth)/
│   │   ├── login/page.tsx           # Sign in (magic link + OAuth)
│   │   └── signup/page.tsx          # Create account
│   ├── (public)/
│   │   ├── merge-pdf/page.tsx
│   │   ├── split-pdf/page.tsx
│   │   ├── compress-pdf/page.tsx
│   │   ├── protect-pdf/page.tsx
│   │   ├── ocr-pdf/page.tsx
│   │   ├── pricing/page.tsx
│   │   ├── api-docs/page.tsx
│   │   ├── enterprise/page.tsx
│   │   ├── security/page.tsx
│   │   └── blog/
│   │       ├── page.tsx             # Blog index
│   │       └── [slug]/page.tsx      # Blog post (SSG)
│   ├── admin/
│   │   ├── layout.tsx               # Admin sidebar + email guard
│   │   ├── page.tsx                 # → redirects to /admin/users
│   │   ├── users/page.tsx
│   │   ├── subscriptions/page.tsx
│   │   ├── analytics/page.tsx
│   │   └── security/page.tsx
│   ├── api/
│   │   ├── auth/callback/route.ts
│   │   ├── billing/
│   │   │   ├── create-checkout/route.ts
│   │   │   └── portal/route.ts
│   │   ├── v1/
│   │   │   ├── compress/route.ts
│   │   │   ├── keys/route.ts
│   │   │   ├── keys/[keyId]/route.ts
│   │   │   ├── jobs/[jobId]/route.ts
│   │   │   └── process/batch/route.ts
│   │   └── webhooks/stripe/route.ts
│   ├── dashboard/
│   │   ├── layout.tsx               # Sidebar + auth check
│   │   ├── page.tsx                 # Overview
│   │   ├── files/page.tsx
│   │   ├── batch/page.tsx
│   │   ├── api-keys/page.tsx
│   │   ├── settings/page.tsx
│   │   └── help/page.tsx
│   ├── layout.tsx                   # Root layout + Toaster
│   ├── page.tsx                     # Landing page
│   ├── not-found.tsx
│   ├── global-error.tsx
│   ├── sitemap.ts
│   └── robots.ts
├── components/
│   ├── ui/                          # Button, Card, Badge, Input, Modal, Progress
│   ├── layout/                      # Navbar, Footer
│   ├── landing/                     # Hero, Features, Pricing, TrustBar
│   ├── tools/                       # PDFMerger, PDFSplitter, PDFCompressor, PDFProtector, FileDropzone
│   └── dashboard/                   # Sidebar
├── lib/
│   ├── pdf/                         # merger.ts, splitter.ts, protector.ts, extractor.ts
│   ├── supabase/                    # client.ts (browser), server.ts (RSC)
│   ├── stripe/                      # client.ts (lazy-initialized)
│   ├── db/schema.sql                # Full Postgres schema (tables, RLS, partitions, trigger)
│   ├── rate-limit.ts                # Upstash sliding-window per tier
│   └── utils.ts                     # cn(), formatBytes(), TIER_LIMITS
├── types/index.ts                   # All shared TypeScript types
└── proxy.ts                         # Next.js middleware (auth + admin guard)
workers/
├── main.py                          # FastAPI worker (Ghostscript + qpdf + Tesseract)
├── requirements.txt
└── Dockerfile
.github/
└── workflows/deploy.yml             # CI: test → Vercel → Cloud Run
env-config.md                        # Environment variable reference
```

---

## User Tiers

| Feature | Free | Pro $9/mo | Team $49/mo |
|---------|------|-----------|-------------|
| Merge files | 5 | 50 | 50 |
| Max file size | 20 MB | 200 MB | 200 MB |
| Split ranges | 3 | Unlimited | Unlimited |
| Compress max | 5 MB | 50 MB | 50 MB |
| Batch | — | 20 files | 200 files |
| API access | — | 1,000 req/mo | 10,000 req/mo |
| OCR | — | 50 pages/mo | 50 pages/mo |
| SSO | — | — | SAML/Google WS |
| Audit logs | — | — | 90 days |
| Ads | Yes | No | No |

---

## Admin Panel

Access `/admin` while signed in with an email listed in `ADMIN_EMAILS`. Unauthorized users are redirected to `/dashboard`.

---

## License

MIT
