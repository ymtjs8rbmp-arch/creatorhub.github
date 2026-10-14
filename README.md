# SpeakIt + CreatorHub — verified email + AI helper

This package adds:
- Email verification before the admin notification is sent.
- A real AI chat endpoint with the OpenAI API key stored only as a Cloudflare Worker secret.
- KV storage for pending applications (20-minute expiry) and basic AI chat rate limiting.
- Privacy-first application fields: creator nickname, not legal full name or exact birthday.

## Important: this needs setup before it works
The frontend alone is not enough. Deploy this through **Cloudflare Workers with Static Assets** (the Worker serves `index.html` and `/api/*`). Do not upload only `index.html` to GitHub Pages and expect verification/AI to work.

### 1. Create Cloudflare KV
1. Create a Cloudflare account and open Workers & Pages.
2. Create a KV namespace named something like `speakit-applications`.
3. Copy its namespace ID into `wrangler.toml`, replacing `REPLACE_WITH_YOUR_CLOUDFLARE_KV_NAMESPACE_ID`.

### 2. Configure email sending (Resend)
1. Create a Resend account.
2. Verify a domain you own and configure its required DNS records. A provider generally will not let you reliably send mail as an arbitrary Gmail address.
3. Choose a verified sender such as `SpeakIt <noreply@your-verified-domain.example>`.
4. In Cloudflare Worker settings, add secrets/environment variables:
   - `RESEND_API_KEY` — secret
   - `EMAIL_FROM` — sender identity allowed by Resend, e.g. `SpeakIt <noreply@yourdomain.com>`
   - `ADMIN_EMAIL` — the admin inbox address
   - `OPENAI_API_KEY` — secret; never put this in `index.html`
   - Optional `OPENAI_MODEL` — defaults to `gpt-4o-mini`

### 3. Deploy
Install Node.js and Wrangler, then from this folder:
```bash
npm install -g wrangler
wrangler login
wrangler kv namespace create APPLICATIONS
```
Use the namespace ID from that command in `wrangler.toml`. Then add secrets:
```bash
wrangler secret put RESEND_API_KEY
wrangler secret put EMAIL_FROM
wrangler secret put ADMIN_EMAIL
wrangler secret put OPENAI_API_KEY
# Optional:
wrangler secret put OPENAI_MODEL
wrangler deploy
```
When prompted, enter the value for each secret. The `EMAIL_FROM` value is a string, but is configured as a Worker secret here for simpler setup. `ADMIN_EMAIL` can also be set as a secret. After deployment, open the Worker URL and test `/api/health` (should return `{"ok":true}`).

If Wrangler reports an issue with the KV binding, make sure the namespace ID is present in `wrangler.toml`. If you created KV through Wrangler, copy the exact ID from its output.

### 4. Test the full flow
1. Open the deployed Worker URL.
2. Submit a test application with an email address you control, and only non-sensitive test data.
3. The page should say “Check your email”; admins should **not** be notified yet.
4. Click the confirmation link in the email. The page should show email confirmed, and only then should the admin inbox receive the application.
5. Open “Ask SpeakIt AI” and ask a simple question such as “What are the application requirements?”

## Security and privacy notes
- The site refuses applications from people who select under 13. Do not try to collect under-13 children's information.
- A checkbox is not strong age verification. If your service has users under 18, ask a trusted adult to review legal/privacy obligations before publishing.
- Keep collected application data minimal. The pending record expires after 20 minutes; verified application details are emailed to the admin and are not stored in KV by this starter.
- AI chat messages are sent to the OpenAI API. Tell visitors this clearly in your privacy notice before launch. Do not ask users to enter sensitive information.
- This starter includes basic rate limiting but not production-grade spam protection, bot challenges, admin review dashboard, or guaranteed email delivery. Review logs and email-provider delivery status.
- Email provider or AI usage may cost money and is subject to each provider's terms and limits.
- Never commit API keys, tokens, `.dev.vars`, or secrets to a public GitHub repository.
- Verification links are single-use and expire after 20 minutes. Admin notification is attempted only after a valid link is used.
- Do not claim an email was sent unless the provider returned success. The verification page handles the normal success path; if admin delivery fails, the verification endpoint shows an error instead of falsely showing success.

## Why not GitHub Pages alone?
GitHub Pages is static hosting. A real confirmation email and AI call require server-side endpoints to keep provider credentials private. This package serves the same static page through a Cloudflare Worker so the `/api` endpoints are available on the same origin.
