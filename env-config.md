## MergePDFly Environment Variables

Copy this file as `.env.local` and populate each value.

```
NEXT_PUBLIC_APP_URL=http://localhost:3000

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_supabase_service_role_key

# Stripe
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_PRO_MONTHLY_PRICE_ID=price_...
STRIPE_PRO_YEARLY_PRICE_ID=price_...
STRIPE_TEAM_MONTHLY_PRICE_ID=price_...

# Upstash Redis
UPSTASH_REDIS_REST_URL=https://your-db.upstash.io
UPSTASH_REDIS_REST_TOKEN=your_upstash_token

# Cloudflare R2
R2_ACCOUNT_ID=your_cloudflare_account_id
R2_ACCESS_KEY_ID=your_r2_access_key_id
R2_SECRET_ACCESS_KEY=your_r2_secret_access_key
R2_BUCKET_NAME=mergepdfly-files
R2_PUBLIC_URL=https://files.mergepdfly.com

# Admin panel access (comma-separated emails)
ADMIN_EMAILS=admin@yourdomain.com

# Cloud Run PDF worker
# Set after first deploy: copy the service URL from Cloud Run console or CI output
CLOUD_RUN_WORKER_URL=https://pdf-workers-xxxx-uc.a.run.app

# Shared secret between Next.js and Cloud Run worker (generate with: openssl rand -hex 32)
# Used in two ways:
#   WORKER_INTERNAL_SECRET  — Next.js → Cloud Run (x-internal-secret header)
#   WORKER_CALLBACK_SECRET  — Cloud Run → Next.js callback (x-worker-secret header)
WORKER_INTERNAL_SECRET=your_random_32_char_hex_secret
WORKER_CALLBACK_SECRET=your_other_random_32_char_hex_secret
```

### GCP Secret Manager setup (required for CI/CD)

Sensitive worker credentials are stored in GCP Secret Manager so they do not appear in `gcloud run services describe` output. Run once per secret (replace `VALUE` with the actual value):

```bash
printf "VALUE" | gcloud secrets create R2_ACCESS_KEY_ID --data-file=-
printf "VALUE" | gcloud secrets create R2_SECRET_ACCESS_KEY --data-file=-
printf "VALUE" | gcloud secrets create UPSTASH_REDIS_REST_URL --data-file=-
printf "VALUE" | gcloud secrets create UPSTASH_REDIS_REST_TOKEN --data-file=-
printf "VALUE" | gcloud secrets create WORKER_INTERNAL_SECRET --data-file=-
printf "VALUE" | gcloud secrets create WORKER_CALLBACK_SECRET --data-file=-
```

Grant the Cloud Run service account access:

```bash
SA="SERVICE_ACCOUNT_EMAIL"  # shown in Cloud Run → Service details → Service account
for secret in R2_ACCESS_KEY_ID R2_SECRET_ACCESS_KEY UPSTASH_REDIS_REST_URL UPSTASH_REDIS_REST_TOKEN WORKER_INTERNAL_SECRET WORKER_CALLBACK_SECRET; do
  gcloud secrets add-iam-policy-binding $secret \
    --member="serviceAccount:$SA" \
    --role="roles/secretmanager.secretAccessor"
done
```
