# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| `main` branch | ✅ |

## Reporting a Vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

Use GitHub's private disclosure system:
👉 **[Report a vulnerability](https://github.com/Priyanshu-byte-coder/devtrack/security/advisories/new)**

This creates an encrypted private thread between you and the maintainer. Your report is never visible to the public until a fix is released.

If the advisory page is unavailable, email **doshipriyanshu3@gmail.com** as a fallback.

Include in your report:
- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (optional but appreciated)

**Response:** Acknowledgement within 48 hours. Fix timeline communicated within 5 business days.

---

## Scope

**In scope:**
- Authentication bypass or session vulnerabilities
- GitHub OAuth token leakage or revocation gaps
- Cross-user data exposure via caching or shared tokens
- SQL injection or Supabase data exposure
- Server-side request forgery (SSRF) via GitHub API proxy
- Missing security headers enabling clickjacking or content injection
- Rate limit exhaustion on shared server tokens

**Out of scope:**
- Issues requiring physical device access
- Social engineering
- Volumetric DoS on free-tier Vercel/Supabase infrastructure

---

## API Response Logging Redaction Standards

DevTrack handles GitHub OAuth data, Supabase session context, webhook payloads, and analytics records that may contain user-specific identifiers. Logs should help maintainers debug production incidents without exposing reusable credentials or enough metadata to reconstruct a private user profile.

### Never log these values

- OAuth access tokens, refresh tokens, authorization codes, session cookies, CSRF tokens, or `NEXTAUTH_SECRET`
- Supabase service-role keys, anon keys, JWTs, database URLs, Redis tokens, webhook secrets, or GitHub personal access tokens
- Raw `Authorization`, `Cookie`, `Set-Cookie`, `X-Hub-Signature-256`, `X-GitHub-Token`, or `x-supabase-auth` headers
- Full webhook bodies, GitHub API responses, or Supabase rows that include emails, provider IDs, tokens, private repository names, or user-owned dashboard data
- Stack traces that include environment variables, request headers, SQL connection strings, or third-party API responses

### Approved logging shape

Prefer structured logs with short, stable fields:

```json
{
  "event": "github_metrics_fetch_failed",
  "route": "/api/github/metrics",
  "status": 502,
  "provider": "github",
  "userIdHash": "sha256:12-char-prefix",
  "requestId": "req_...",
  "retryable": true
}
```

Use a one-way hash or short prefix when an identifier is needed for correlation. Do not log raw user IDs, emails, repository tokens, or full provider payloads. For GitHub repository names, prefer counts or visibility-neutral labels unless the route already returns the same public repository name to the authenticated user.

### Redaction checklist for API handlers

- Log the event name, route, status code, provider, and request ID before adding any response details.
- Redact secrets before logging errors returned from GitHub, Supabase, Redis, Groq, or webhook verification code.
- Replace token-like values with `[REDACTED]`, not partial real tokens.
- Convert user identifiers to a deterministic hash when cross-request correlation is required.
- Trim arrays and nested objects to counts or whitelisted keys before logging.
- Keep raw response bodies out of logs; attach a sanitized error code or category instead.
- Review new logging in PRs by searching for `console.`, `logger.`, `Authorization`, `Cookie`, `token`, `secret`, `key`, and `password`.

### Examples

Unsafe:

```ts
console.error("GitHub response failed", { headers, body, token: accessToken });
```

Safe:

```ts
console.error("GitHub response failed", {
  event: "github_response_failed",
  status,
  requestId,
  retryable: status >= 500,
});
```

If a maintainer needs full provider payloads during incident response, capture them through a private, access-controlled channel and delete them after the investigation window. Public CI logs, Vercel logs, and issue comments must only contain redacted output.

---

## Points & Recognition (GSSoC)

Security fixes are treated as **`level:critical`** — highest point tier in the GSSoC scoring system. A private advisory serves as the issue record; no public issue is required. Points are awarded on merge based on impact and fix quality.

---

## Coordinated Disclosure

Once a fix ships, a summary is published in [GitHub Security Advisories](https://github.com/Priyanshu-byte-coder/devtrack/security/advisories). Reporters are credited by name unless they request anonymity.

---

## Row Level Security (RLS)

DevTrack uses Supabase with Row Level Security on all user-data tables.

| Table | RLS | Policies |
|-------|-----|---------|
| `users` | ✅ | SELECT, UPDATE own row only |
| `goals` | ✅ | SELECT, INSERT, UPDATE, DELETE own rows only |
| `metric_snapshots` | ✅ | SELECT, INSERT, DELETE own rows only |

- All RLS policies match against `auth.uid()`
- `supabaseAdmin` (service role key) is server-side only, never exposed to clients
- The anon key has no direct table access by default
