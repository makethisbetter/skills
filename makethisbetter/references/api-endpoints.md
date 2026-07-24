# MakeThisBetter CLI Reference

## Install

```bash
npm install -g @makethisbetter/cli
```

The CLI binary is `makethisbetter` — a precompiled Go binary; the npm package
is a thin installer (Node is only needed to run npm itself). Alternative:
download the binary for your platform from
https://github.com/makethisbetter/cli/releases.

## Config

Saved at `~/.makethisbetter/config.json` with mode `600`:

```json
{
  "api_token": "token_...",
  "api_url": "https://makethisbetter.dev/api/v1",
  "account_id": "acct_...",
  "user_email": "dev@example.com"
}
```

## Commands

### Login

```bash
makethisbetter login
makethisbetter login --email dev@example.com --otp 123456
makethisbetter login --token token_xxx --api-url https://makethisbetter.dev/api/v1
makethisbetter login --account-id acct_xxx
```

Interactive mode prompts for email and verification code.
`--token` saves an existing token without OTP.

### Feedback list

```bash
makethisbetter feedback list --project acme
makethisbetter feedback list --project acme --status received
makethisbetter feedback list --project acme --label Safari
makethisbetter feedback list --project acme --priority high
makethisbetter feedback list --project acme --sort priority
makethisbetter feedback list --project acme --json
```

Filters: `--status`, `--label`, `--priority`. `--project` is required. AI selects
labels from the system-managed pool; the CLI only reads and filters them.
Sort: `priority`, `created`, `updated`.

### Feedback show

```bash
makethisbetter feedback show <handle/FB-n> --md    # server-rendered markdown, cheapest tokens
makethisbetter feedback show acme/FB-1 --json
```

Displays full feedback detail: description, AI analysis, reporter, page URL,
console errors, target element, and timeline. `--md` prints the
server-rendered markdown (single source of truth); `--json` includes it as the
`markdown` field.

### Project list / show / create

```bash
makethisbetter project list [--json]
makethisbetter project show <handle> [--json]
makethisbetter project create <name> --handle <handle> [--domain example.com] [--json]
```

`show` returns `api_key` (the widget `projectKey`) and `board_url`;
`signing_secret` is included only when the logged-in user is an account
admin. `create` requires account admin (403 otherwise) and returns the new
project including `api_key` and `signing_secret`.

### Feedback pick / resolve / dismiss

```bash
makethisbetter feedback pick <handle/FB-n>                        # status -> in_progress
makethisbetter feedback resolve <handle/FB-n> [--pr <url>]        # closed + close_reason shipped
makethisbetter feedback dismiss <handle/FB-n> --reason not_planned  # closed + close_reason not_planned
makethisbetter feedback dismiss <handle/FB-n> --reason duplicate    # closed + close_reason duplicate
```

All three accept `--json` for machine-readable output. Status values are
`received`, `in_progress`, `pending_release`, `closed`; `shipped` /
`not_planned` / `duplicate` are `close_reason` values on `closed`.

## API Endpoints

Default base URL: `https://makethisbetter.dev/api/v1`

### Agent Registration

```http
POST /api/v1/agent_registration
Content-Type: application/json

{ "email": "dev@example.com" }
```

Response `200`: `{ "registration_token": "<jwt>", "expires_in": 300 }`

### Verify OTP

```http
POST /api/v1/agent_registration/verify
Content-Type: application/json

{ "registration_token": "<jwt>", "otp": "123456" }
```

Response `200`:

```json
{
  "user": { "id": "user_...", "email": "dev@example.com" },
  "account": { "id": "acct_...", "name": "Dev's Team" },
  "api_token": { "token": "token_...", "name": "Agent CLI" }
}
```

### List / show / create projects

```http
GET /api/v1/projects
Authorization: Bearer token_...
```

Returns `[{ "id": "project_...", "name": "...", "domain": null, "feedback_visibility": "...", "feedbacks_count": 0, ... }]`.

```http
GET /api/v1/projects/:id
Authorization: Bearer token_...
```

Adds `api_key` (the widget `projectKey`), `board_url`,
`enforce_identity_verification`, and — for account admins only —
`signing_secret`.

```http
POST /api/v1/projects
Authorization: Bearer token_...
Content-Type: application/json

{ "project": { "name": "My App", "handle": "my-app", "domain": "example.com" } }
```

Requires account admin (`403` otherwise). Response `201` matches the show
shape including `api_key` and `signing_secret`.

### List feedback

```http
GET /api/v1/projects/acme/feedbacks?status=received
Authorization: Bearer token_...
```

Query params: `status`, `label`, `priority`, `account_id`. The response exposes
`labels` as an array of AI-managed project label names.

### Show feedback

```http
GET /api/v1/projects/acme/feedbacks/1
Authorization: Bearer token_...
```

### Update feedback

```http
PATCH /api/v1/projects/acme/feedbacks/1
Authorization: Bearer token_...
Content-Type: application/json

{ "feedback": { "status": "in_progress" } }
```

Accepted fields: `status`, `priority`, `close_reason`, `pr_url`, and
`ai_structured_summary`. Labels are read-only through this API. For transition
compatibility, old clients may still send `labels.close_reason` and
`labels.pr_url`; other nested label values are ignored.

## Error Codes

- `401`: invalid or expired token.
- `404`: feedback not found.
- `422`: invalid parameters.
- `429`: rate limited.
