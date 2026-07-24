---
name: makethisbetter
description: Set up, authenticate, and manage MakeThisBetter feedback from Claude Code. Use when the user runs /makethisbetter followed by setup, login, signup, list, pick, dismiss, resolve, or any feedback management command. Also triggers on MakeThisBetter setup, listing pending feedback, choosing feedback to fix, skipping feedback, marking feedback resolved, or linking a PR to feedback.
---

# MakeThisBetter

## Overview

Install the CLI, authenticate, and manage user feedback — all without leaving
Claude Code. One skill, one namespace.

## Commands

Dispatch based on the first argument. **No argument → default to `list`** (show
received feedback for the current project).

| Invocation                                         | Action                              |
|----------------------------------------------------|-------------------------------------|
| `/makethisbetter` *(no args)*                      | **Default**: list received feedback for the current project (same as `list`) |
| `/makethisbetter setup`                            | Install CLI, log in, resolve project, configure and install the widget |
| `/makethisbetter signup`                           | Create account via email OTP        |
| `/makethisbetter login`                            | Log in via email OTP (same as signup) |
| `/makethisbetter list`                             | List received feedback              |
| `/makethisbetter pick <handle/FB-n>`                        | Read context and start working      |
| `/makethisbetter dismiss <handle/FB-n> --reason not_planned` | Decline a feedback item (`not_planned`/`duplicate`) |
| `/makethisbetter resolve <handle/FB-n>`                     | Mark done and link PR               |
| `/makethisbetter setup-auth`                       | Guide identity verification setup   |

---

## Account Commands

### /makethisbetter setup

Run these steps in order. Stop and report if any step fails.

1. Check if the CLI is installed:

```bash
npm list -g @makethisbetter/cli 2>/dev/null
```

2. If not installed, install it:

```bash
npm install -g @makethisbetter/cli
```

If the npm install fails, fall back to downloading the prebuilt binary from
GitHub Releases (pick the asset matching the platform, e.g.
`makethisbetter-darwin-arm64`):

```bash
curl -sL "https://github.com/makethisbetter/cli/releases/latest/download/makethisbetter-$(uname -s | tr '[:upper:]' '[:lower:]')-$(uname -m | sed 's/x86_64/x64/;s/aarch64/arm64/')" -o /usr/local/bin/makethisbetter && chmod +x /usr/local/bin/makethisbetter
```

3. Check if already logged in:

```bash
cat ~/.makethisbetter/config.json 2>/dev/null
```

4. If `api_token` is missing or the file does not exist, run login:

```bash
makethisbetter login
```

5. Detect the project framework and recommend a widget install method:

```bash
# Check these files to determine the framework:
# - config/importmap.rb          -> Rails 8 (Import Map)
# - Gemfile with "rails"         -> Rails (check version for 7 vs 8)
# - package.json with "next"     -> Next.js / React
# - package.json with "nuxt"     -> Nuxt / Vue
# - package.json with "svelte"   -> SvelteKit / Svelte
# - package.json with "vue"      -> Vue
# - package.json with "react"    -> React
# - composer.json                -> PHP (CDN)
# - wp-content/ or style.css with "Theme Name" -> WordPress (CDN)
# - theme.liquid or *.liquid     -> Shopify (CDN)
# - *.jsp or pom.xml             -> Java/JSP (CDN)
# - None of the above            -> Plain HTML (CDN)
```

Use this detection order (first match wins):

| Detected | Recommendation |
|---|---|
| `config/importmap.rb` exists | `pin "makethisbetter"` in `config/importmap.rb`, import in `application.js` |
| `Gemfile` contains `rails` + `package.json` exists | `npm i makethisbetter`, import in `application.js` |
| `package.json` contains `next` | `npm i makethisbetter`, init in `layout.tsx` or `_app.tsx` `useEffect` |
| `package.json` contains `nuxt` | `npm i makethisbetter`, init as Nuxt plugin |
| `package.json` contains `svelte` | `npm i makethisbetter`, init in `+layout.svelte` `onMount` |
| `package.json` contains `vue` | `npm i makethisbetter`, init in `App.vue` `onMounted` |
| `package.json` contains `react` | `npm i makethisbetter`, init in root component `useEffect` |
| `composer.json` or `wp-content/` | CDN `<script>` tag in `header.php` |
| `*.liquid` files | CDN `<script>` tag in `theme.liquid` |
| Anything else | CDN `<script src="https://unpkg.com/makethisbetter@1/dist/makethisbetter.js">` |

6. Read the project README to understand context:

```bash
cat README.md
```

7. Resolve the project (see "Project Resolution" below) — this yields the
   `projectKey` without ever opening the dashboard.

8. Run the Widget Configuration Interview (see below) to infer the remaining
   `init()` options from the codebase.

9. Apply the installation: edit the user's code using the install method from
   step 5 and the configuration from steps 7–8.

10. Verify at the HTTP layer (see "Verify the Installation" below). Do NOT
    submit a test feedback.

### Project Resolution

Get the project and its key entirely from the CLI:

```bash
makethisbetter project list --json
```

- **Exactly one project** → use it. Mention which one in the final summary.
- **Multiple projects** → ask the user which one to use (this counts toward
  the two-question budget). Recommend the best match by name/domain.
- **No projects** → ask the user for a project name (recommend the repo or
  site name), then create it:

```bash
makethisbetter project create "My App" --handle my-app --json
```

Then fetch the key (and signing secret, if the user is an account admin):

```bash
makethisbetter project show <handle> --json
```

The `api_key` field is the widget `projectKey`. Never send the user to the
dashboard for keys.

### Widget Configuration Interview

Rules — follow them strictly:

- Infer every option from the codebase FIRST. Never ask about something the
  code already answers.
- Ask the user AT MOST 2 questions total during setup, one at a time. Budget:
  one for project pick/name (only when needed, see Project Resolution), one to
  confirm the inferred configuration.
- Every question and the confirmation must include your recommendation with
  evidence: "I found X, so I recommend Y."
- Ask in the user's language.

Inference table:

| Option | Codebase signals | Recommendation logic |
|---|---|---|
| `locale` | `config/locales/`, next-intl / vue-i18n config, `<html lang>`, README language | Single-language site → set the matching locale (`en`, `zh-CN`, `ja`, `ko`, `es`, `fr`, `de`). Multi-language site → interpolate the server locale, e.g. Rails `locale: '<%= I18n.locale %>'` |
| `theme` | Tailwind `dark:` classes, `prefers-color-scheme` CSS, theme toggle code | Site supports dark mode → keep default `auto` (omit). Light-only site → `'light'` |
| `position` | Existing floating widgets in the bottom-right (Intercom, Crisp, Chatwoot, Drift script tags) | Conflict → `'left'`. Otherwise keep default `'right'` (omit) |
| `tabText` | Brand tone in README / marketing copy | Keep the built-in localized label unless branding clearly wants otherwise |
| `frustrationDetection` | none | Keep default (on). In the confirmation, explain in one line: proactively prompts users after rage clicks, rapid navigation, form failures, or error pages |
| `user` identity | Auth framework (Devise, NextAuth, Clerk, session middleware) | Site has logged-in users → suggest passing `user: { id, email, name }`; for spoof-proof identity point to `/makethisbetter setup-auth` |
| `apiUrl` | none | Only for self-hosted backends; never ask proactively |

Present the full inferred configuration as ONE confirmation question — each
option with its value and the evidence — and let the user confirm or adjust.

Then write the final init with only non-default values, e.g.:

```js
MakeThisBetter.init({
  projectKey: "mtb_proj_xxx",
  locale: "zh-CN",
  position: "left"
})
```

### Verify the Installation

HTTP-layer checks only — never submit a test feedback (it would pollute the
project's data and consume AI triage):

1. CDN reachable: `curl -sI https://unpkg.com/makethisbetter@1` returns a 2xx/3xx.
2. The rendered page contains the widget script and the `MakeThisBetter.init`
   call (for server-rendered apps, `curl` the local dev server and grep).
3. If a dev server and browser tooling are available, confirm the page loads
   with no console errors from the widget.

Finish by summarizing: project used, install method, final `init()` options
with the evidence for each, and how to run `/makethisbetter list` once
feedback arrives.

### /makethisbetter signup and /makethisbetter login

Both use the same OTP flow. The backend uses find-or-create semantics.

```bash
makethisbetter login
```

The command prompts for email, sends a verification code, then prompts for the
6-digit code. On success it saves the API token to `~/.makethisbetter/config.json`.

For automation (skipping interactive prompts):

```bash
makethisbetter login --email dev@example.com --otp 123456
```

To save an existing token directly:

```bash
makethisbetter login --token token_xxx
```

### /makethisbetter setup-auth

Guide the developer through setting up identity verification so feedback is
linked to authenticated users.

1. Verify the CLI is logged in:

```bash
makethisbetter info
```

If the command fails, prompt the user to run `/makethisbetter setup` first.
Then read the project's Signing Secret from the CLI:

```bash
makethisbetter project show <handle> --json
```

The `signing_secret` field is only returned when the logged-in user is an
account admin. If it is absent, tell the user an account admin must run this
step (or share the secret with them).

2. Show code snippets for generating a JWT based on the project's backend
   framework. Detect the framework the same way `/makethisbetter setup` does
   (check for `Gemfile`, `package.json`, `requirements.txt`, etc.) and show the
   matching snippet. If the framework is unclear, show all four.

**Rails**

```ruby
# Gemfile: gem "jwt"
token = JWT.encode({ sub: current_user.id, email: current_user.email }, signing_secret, "HS256")
```

**Express**

```js
// npm i jsonwebtoken
const jwt = require("jsonwebtoken");
const token = jwt.sign({ sub: user.id, email: user.email }, signingSecret);
```

**Next.js (API route)**

```ts
// npm i jsonwebtoken
import jwt from "jsonwebtoken";
const token = jwt.sign({ sub: user.id, email: user.email }, process.env.MAKETHISBETTER_SIGNING_SECRET!);
```

**Python (Flask / Django / FastAPI)**

```python
# pip install pyjwt
import jwt
token = jwt.encode({"sub": user_id, "email": email}, signing_secret, algorithm="HS256")
```

3. Tell the developer to pass the token when initializing the widget:

```js
MakeThisBetter.init({
  projectKey: "mtb_proj_xxx",
  userToken: token
});
```

Where `token` is the JWT generated server-side and injected into the page (via a
meta tag, inline script variable, or API response).

4. Remind the developer that feedback submitted with a valid `userToken` will
   show the verified user identity in the dashboard.

---

## Feedback Commands

### Input Parsing

All feedback commands expect a `handle/FB-n` reference (e.g. `feedback/FB-5`).
When the user provides something else, normalize it **before** calling any MCP
tool or CLI command:

| User input | How to normalize |
|---|---|
| `feedback/FB-5` | Already correct — use as-is |
| `FB-5` or `fb-5` (bare ID, no handle) | Resolve the project handle first: call `list_projects` (MCP) or `makethisbetter project list --json` (CLI). One project → use its handle. Multiple → ask the user which one |
| `https://makethisbetter.dev/feedback/fb-5` | Dashboard URL — extract path segments: handle = `feedback`, id = `fb-5` → `feedback/FB-5` |
| `https://makethisbetter.dev/en/feedback/fb-5` | Localized dashboard URL — skip the locale segment (`en`, `zh-CN`, etc.), then same as above |
| `https://{handle}.makethisbetter.dev/fb-5` | Subdomain board URL — subdomain is the handle, path is the id → `{handle}/FB-5` |

Rules:

1. **Uppercase the ID**: `fb-5` → `FB-5` (the MCP tool requires uppercase).
2. **Dashboard URL path**: `/{optional-locale}/{project-handle}/{fb-id}`. A locale
   segment is 2-letter or 2+2 format (e.g. `en`, `zh-CN`) — skip it.
3. **Never guess the handle** — if you can't extract it from the URL and there are
   multiple projects, ask the user.

### Prerequisites

The `makethisbetter` CLI must be installed and logged in. Run `/makethisbetter setup`
first if not configured.

### /makethisbetter list

#### Project auto-detection

Resolve the project handle automatically — never ask the user or list all
projects blindly:

1. Call `list_projects` (MCP) or `makethisbetter project list --json` (CLI).
2. **One project** → use it.
3. **Multiple projects** → match by the current working directory:
   - Compare each project's `domain` against the repo/directory name,
     `package.json` `name` or `homepage`, or the site domain in any
     framework config (e.g. `next.config.js`, `nuxt.config.ts`, `.env`).
   - First match wins. If no match, ask the user which one.
4. **No projects** → tell the user to run `/makethisbetter setup` first.

Then list feedback for that single project:

```bash
makethisbetter feedback list --project <handle> --status received
```

Optional filters:

```bash
makethisbetter feedback list --project <handle> --status in_progress
makethisbetter feedback list --project <handle> --label Safari
makethisbetter feedback list --project <handle> --priority high
makethisbetter feedback list --project <handle> --sort priority
```

Add `--json` for machine-readable output.

### /makethisbetter pick

#### Required context inspection

Before interpreting, picking, or editing, inspect **all available original
context**, not only the description or AI summary:

1. Read the text detail first:

```bash
makethisbetter feedback show <handle/FB-n> --md
```

2. Read `--json` and check the raw fields: `page_url`, `target_element`,
   `console_errors`, browser/OS, `screenshot_attached`, `recording_attached`,
   `recording_url`, and AI analysis. AI analysis is a lead, not a substitute for
   the original evidence.
3. When `screenshot_attached` is true, proactively open the authenticated
   dashboard feedback page with available browser tooling and inspect the full
   screenshot, annotation, and surrounding page context. Zoom or crop as needed
   to identify the exact element. **Do not ask the user what the screenshot
   shows before trying to inspect it yourself.**
4. When `recording_attached` is true, watch the recording and note the action
   sequence. Read breadcrumbs and console/network errors when available.
5. Reconcile conflicts in this order: original screenshot/recording and raw
   telemetry → user description and clarification → AI summary. State any
   remaining uncertainty explicitly; never invent missing context.

If an attachment is flagged but the CLI/MCP response omits its URL, use the
logged-in dashboard page to view it. An attachment is "blocked" only after you
have attempted the dashboard detail page with available browser tooling and
can name the specific authorization, navigation, or rendering error. Record
that error, then ask for only the missing evidence.

**Attachment gate:** when `screenshot_attached` or `recording_attached` is true,
do not interpret the request, run `feedback pick`, or edit code until you have
either inspected that attachment or recorded the specific blocking error above.
A vague description, AI summary, thumbnail, or missing URL does not satisfy this
gate.

Only after this inspection:

6. Move feedback to in_progress:

```bash
makethisbetter feedback pick <handle/FB-n>
```

7. Start editing code based on the verified user scenario and evidence.

### /makethisbetter dismiss

Decline a feedback item with a close reason (`not_planned` by default, or
`duplicate`):

```bash
makethisbetter feedback dismiss <handle/FB-n> --reason not_planned
```

Confirm the dismissal to the user by showing the feedback ID and reason.

### /makethisbetter resolve

1. Auto-detect the PR URL from the current git branch:

```bash
gh pr view --json url -q .url 2>/dev/null
```

2. Mark the feedback as shipped (pass the PR URL when one was detected or
   provided):

```bash
makethisbetter feedback resolve <handle/FB-n> --pr <url>
```

3. Report the result to the user. The CLI stores `close_reason: shipped`
   automatically and the reporter gets notified.

If the user provides an explicit `--pr <url>`, use that instead of
auto-detecting.

---

## Workflow

Use `/makethisbetter list` to find received feedback, `/makethisbetter pick <handle/FB-n>` to
read context and start working, then `/makethisbetter resolve <handle/FB-n>` to mark it shipped.
Use `/makethisbetter dismiss <handle/FB-n> --reason not_planned` to decline items you choose not to
address.

## Widget Installation

Choose the install method based on the project's framework.

| Framework | Method | How |
|---|---|---|
| Plain HTML / PHP / JSP | CDN script tag | `<script src="https://unpkg.com/makethisbetter@1/dist/makethisbetter.js"></script>` then `<script>MakeThisBetter.init({...})</script>` |
| Rails 8 (Import Map) | importmap pin | `pin "makethisbetter"` in `config/importmap.rb`, `import "makethisbetter"` in `application.js` |
| Rails 7 / esbuild | npm install | `npm i makethisbetter`, import in `application.js` |
| Next.js / React | npm install | `npm i makethisbetter`, init in `_app.tsx` or `layout.tsx` inside `useEffect` |
| Vue / Nuxt | npm install | `npm i makethisbetter`, init in `App.vue` `onMounted` or as a Nuxt plugin |
| Svelte / SvelteKit | npm install | `npm i makethisbetter`, init in `+layout.svelte` `onMount` |
| WordPress | CDN script tag | Add to `header.php` or via a custom-header plugin |
| Shopify | CDN script tag | Add to `theme.liquid` before `</head>` |

### Init call

All methods end with the same init:

```js
MakeThisBetter.init({ projectKey: "mtb_proj_xxx" });
```

Get the project key with `makethisbetter project show <handle> --json` (the
`api_key` field), and the full option set from the Widget Configuration
Interview in `/makethisbetter setup`.

## Status Flow

```
received -> in_progress -> shipped (closed)
                       -> not_planned (closed)
                       -> duplicate (closed)
```

- `received`: new feedback ready for triage.
- `in_progress`: a developer or agent picked it up.
- `pending_release`: a fix exists and is waiting for release.
- `closed`: the fix shipped, was declined, or was marked duplicate.

`shipped`, `not_planned`, and `duplicate` are `close_reason` values on the
`closed` status: `feedback resolve` sets `shipped`, `feedback dismiss --reason`
sets `not_planned` or `duplicate`.

## Reference

Read `references/api-endpoints.md` when you need endpoint details, CLI options,
config file format, or error handling behavior.
