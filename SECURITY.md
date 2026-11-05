# Security Policy & Architecture

**Project:** Manipal University Jaipur — Patent Application & Disclosure Portal  
**Compliance Standard:** OWASP ASVS L2 • DPDP 2025 Architecture • Indian Patents Act 1970 (Section 39 Compliance)

---

## 1. Threat Model & Institutional Context

The MUJ Patent Application Portal safeguards early-stage invention disclosures, prior-art searches, and technical drawings before formal statutory patent filing with the Indian Patent Office (IPO / CGPDTM).

Under Indian Patent Law (Patents Act 1970), India operates on an absolute first-to-file novelty principle with no general grace period. Public disclosure or unauthenticated data leakage (accidental exposure of invention claims, blueprints, or source code) prior to filing destroys absolute novelty and irrevocably invalidates patentability.

Accordingly, the system operates under a **Confidential by Default** security architecture:
1. **Zero Public Disclosure:** Unauthenticated visitors cannot browse disclosures, claims, or inventor identities.
2. **Student Isolation:** Strict multi-tenant row-level security prevents students from viewing or enumerating other students' disclosures.
3. **Confidentiality Shield:** Super admins (IT governance) oversee pipelines and accounts but are strictly barred at the database level from accessing invention descriptions, claims, or technical drawings.
4. **Data Minimization:** Transactional emails never contain invention text or claims.

---

## 2. Secrets Management Policy

### Client-Side vs. Server-Side Segregation

| Environment Variable | Visibility | Scope & Storage | Purpose |
|---|---|---|---|
| `VITE_SUPABASE_URL` | **Public** | Client bundle (`.env`) | Connects frontend to Supabase PostgREST and Auth APIs. Protected by PostgreSQL Row Level Security (RLS). |
| `VITE_SUPABASE_ANON_KEY` | **Public** | Client bundle (`.env`) | PostgREST anonymous client gateway key. Subject to RLS policies. |
| `VITE_PORTAL_BASE_URL` | **Public** | Client bundle (`.env`) | Canonical institutional portal URL for CORS and redirects. |
| `VITE_TURNSTILE_SITE_KEY` | **Public** | Client bundle (`.env`) | Cloudflare Turnstile public site key for bot verification. |
| `SUPABASE_SERVICE_ROLE_KEY` | **SECRET** | Supabase Edge Secrets (`Deno.env`) | Bypasses RLS. Used exclusively within Edge Functions (`reset-authenticator`) for administrative factor resets. **Never expose to client.** |
| `MS_TENANT_ID`, `MS_CLIENT_ID` | **Internal** | Supabase Edge Secrets (`Deno.env`) | Microsoft 365 Azure Entra ID app credentials. |
| `MS_CLIENT_SECRET` | **SECRET** | Supabase Edge Secrets (`Deno.env`) | OAuth 2.0 client secret for university Exchange Online mailbox delivery. |
| `SEND_EMAIL_HOOK_SECRET` | **SECRET** | Supabase Edge Secrets (`Deno.env`) | Webhook signature secret verifying Auth email requests from Supabase GoTrue. |
| `SUPABASE_AUTH_CAPTCHA_SECRET`| **SECRET** | Supabase Auth Settings | Cloudflare Turnstile secret key verifying client tokens on login/signup. |
| `RESEND_API_KEY` | **SECRET** | Supabase Edge Secrets (`Deno.env`) | Transactional email delivery API key (if Resend provider is active). |

### Rules for Secrets:
- **Never commit `.env` files:** Protected by [`.gitignore`](.gitignore) (`.env*` wildcard with `!.env.example` exception).
- **Startup Validation:** [`src/config/env.ts`](src/config/env.ts) validates that production builds fail fast if critical environment variables are absent or populated with placeholder values.
- **Continuous Secret Scanning:** GitHub Actions CI executes Gitleaks on every commit and pull request.

---

## 3. Key Rotation Runbooks

### A. Rotating Microsoft 365 Client Secret (`MS_CLIENT_SECRET`)
Microsoft Entra ID client secrets expire at a maximum duration of 24 months.
1. Sign in to the **Azure Portal** (`portal.azure.com`) as Global or Application Administrator.
2. Navigate to **Microsoft Entra ID > App registrations > MUJ Patent Portal Mailer**.
3. Go to **Certificates & secrets > Client secrets > New client secret**.
4. Set description `Mailer Rotation YYYY` and expiry (recommended 12-24 months).
5. Copy the new secret **Value** immediately (value is hidden once navigating away).
6. Update Supabase Edge Secrets via CLI:
   ```bash
   npx supabase secrets set --project-ref <project-ref> MS_CLIENT_SECRET="<new-secret-value>"
   ```
7. Verify dispatch with a test registration email.
8. Delete the old secret in Azure Portal once verified.

### B. Rotating Cloudflare Turnstile Secret (`SUPABASE_AUTH_CAPTCHA_SECRET`)
1. Log in to the **Cloudflare Dashboard > Turnstile**.
2. Select the widget for `patents.muj.manipal.edu`.
3. Under Settings, generate or retrieve a new Secret Key.
4. In Supabase Dashboard: **Authentication > Bot and Abuse Protection > Turnstile**.
5. Paste the new Secret Key and save.
6. Does not invalidate existing active sessions.

### C. Rotating Supabase Service Role Key (`SUPABASE_SERVICE_ROLE_KEY`)
1. In Supabase Dashboard: **Project Settings > API > JWT Settings**.
2. Generate a new JWT secret.
3. Copy the updated `service_role` secret.
4. Update the Edge Function secrets:
   ```bash
   npx supabase secrets set --project-ref <project-ref> SUPABASE_SERVICE_ROLE_KEY="<new-key>"
   ```
5. Note: Rotating the JWT secret terminates all active user sessions, requiring users to log in again.

---

## 4. Defense-in-Depth Implementation

### 1. Authentication & Domain Restriction
- **Domain Enforcement:** Both frontend form validation and PostgreSQL trigger (`enforce_university_email_domain`) strictly reject non-`@muj.manipal.edu` addresses.
- **Mandatory Email Verification:** Accounts cannot log in or file applications until the 6-digit email OTP (dispatched via Microsoft Graph) is verified.
- **Multi-Factor Authentication (TOTP):** RFC 6238 TOTP required for administrative actions (`is_admin()` and `is_super_admin()`).
- **Anti-Enumeration:** Sign-up, login, forgot password, and resend endpoints return identical generic wording whether or not an account exists.
- **Rate Limiting:** Supabase Auth rate limits are coupled with dynamic client countdown timers in the UI.

### 2. Authorization & Data Isolation
- **Row-Level Security (RLS):** All database tables have RLS enabled with default-deny policies.
- **Ownership Verification:** Applications are readable and editable only by their `owner_id` (or authorized patent coordinators).
- **Tamper-Proof Fields:** Status transitions, sequential reference numbers (`MUJ-IPR-YYYY-XXXX`), and version snapshots can only be created via secure PL/pgSQL functions (`SECURITY DEFINER` with validated caller IDs).

### 3. File Upload Safety
- **Dual-Tier Validation:** Checks client-reported MIME type AND strictly verifies file extensions against `['.pdf', '.doc', '.docx', '.png', '.jpg', '.jpeg']`.
- **Filename Sanitization:** Filenames are stripped of special characters, sanitized to alphanumeric characters, and truncated to prevent path traversal or executable masquerading.
- **Private Storage:** Files reside in private Supabase Storage buckets accessible solely via short-lived (15-minute) signed URLs.

### 4. HTTP Headers & Content Security Policy (CSP)
- `Content-Security-Policy`: Restricts scripts, styles, frames, and connect endpoints to self, Cloudflare Turnstile (`challenges.cloudflare.com`), Google Fonts, and the designated Supabase domain.
- `X-Content-Type-Options: nosniff`: Prevents browser MIME-type sniffing.
- `Referrer-Policy: strict-origin-when-cross-origin`: Restricts sensitive path information in outgoing referrers.

---

## 5. Security Vulnerability Reporting

If you identify a potential security vulnerability within this repository or the live portal:
1. **Do not create a public GitHub Issue.**
2. Send an email to the institutional security contact: `ipr.cell@muj.manipal.edu` with subject `[SECURITY DISCLOSURE] Patent Portal`.
3. Provide reproduction steps, potential impact, and your contact details for coordinated disclosure.
4. The IPR Cell and IT Systems Directorate will acknowledge receipt within 48 hours and coordinate remediation.
