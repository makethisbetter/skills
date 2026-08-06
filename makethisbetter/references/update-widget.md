# Update Widget

Run this workflow for `/makethisbetter update-widget`. Stop and report the exact
failed step if any command fails.

## 0. Read the Product README

Fetch and read the Make This Better skills repository README before proceeding:

```bash
curl -fsSL https://raw.githubusercontent.com/makethisbetter/skills/main/README.md
```

It contains the full commands reference, example session, and installation
guide. This context is required to make informed decisions during the update.

## Hard Rules

- Never read or print `~/.makethisbetter/config.json`; it contains a bearer
  token. Use `makethisbetter info` for a masked local status.
- Only ask about **new** configuration options the project has not yet set.
  Already-configured options are kept as-is unless the user explicitly asks to
  change them.
- Keep the Signing Secret out of source code, browser code, chat, summaries,
  and logs.
- Never submit test Feedback.

## 1. Verify Prerequisites

Confirm the CLI is available and authenticated:

```bash
command -v makethisbetter && makethisbetter project list --json >/dev/null
```

If either fails, tell the user to run `/makethisbetter setup` first and stop.

## 2. Locate the Current Widget Installation

Search the codebase for the existing `MakeThisBetter.init(` call. Check these
locations in order:

1. `app/javascript/` (Rails with JS bundling)
2. `app/assets/javascripts/` or `app/assets/builds/` (Rails legacy)
3. `app/views/layouts/` (inline `<script>` or ERB)
4. `src/` or `pages/` or `components/` (SPA frameworks)
5. `*.html`, `*.liquid`, `*.php`, `*.jsp` (CDN installs)

```bash
grep -rn "MakeThisBetter.init(" --include="*.js" --include="*.ts" --include="*.erb" --include="*.html" --include="*.vue" --include="*.svelte" --include="*.tsx" --include="*.jsx" --include="*.liquid" --include="*.php" .
```

If no `init(` call is found, tell the user the widget is not installed and
suggest `/makethisbetter setup`. Stop.

Record the file path and the full `init()` configuration object.

## 3. Parse Current Configuration

Extract every option key from the existing `init()` call. The canonical option
list for comparison is:

| Option | Type | Default |
|---|---|---|
| `projectKey` | string | *(required)* |
| `entryMode` | `'button'` \| `'api'` | `'button'` |
| `locale` | string | `document.documentElement.lang ?? 'en'` |
| `position` | `'left'` \| `'right'` | `'right'` |
| `theme` | `'light'` \| `'dark'` \| `'auto'` | `'auto'` |
| `apiUrl` | string | `'https://makethisbetter.dev/api/v1'` |
| `frustrationDetection` | boolean | `true` |
| `tabText` | string | SDK localized default |
| `brandColors` | `{ primary, hover, active, onPrimary }` | SDK default |
| `userToken` | string | `undefined` |
| `userTokenFn` | `() => Promise<string>` | `undefined` |
| `user` | `{ id?, email?, name? }` | `undefined` |

Build two sets:
- **configured**: keys present in the current `init()` call
- **available**: keys from the canonical list above that are NOT in configured
  (excluding `projectKey` which is always present, and `apiUrl` which is only
  for self-hosted)

If **available** is empty, report that the widget configuration is up to date
and skip to Step 5.

## 4. Infer and Confirm New Options

For each option in **available**, infer a recommendation from the codebase
using the same signals as `setup.md` Step 5:

| Option | Signals | Recommendation |
|---|---|---|
| `locale` | `config/locales/`, i18n libraries, `<html lang>`, README language | Single language: set that locale. Multiple: interpolate the server/client locale |
| `theme` | theme toggle, Tailwind `dark:` classes, `prefers-color-scheme` | Dark mode support: keep default `auto`. Light-only: set `light` |
| `position` | Intercom, Crisp, Chatwhat, Drift, or another bottom-right control | Conflict: `left`. Otherwise keep default `right` |
| `tabText` | established product wording | Keep the localized default unless the product has a clearly different label |
| `brandColors` | An explicit semantic `primary` / `hover` / `active` / `onPrimary` group in CSS custom properties, Tailwind theme colors, design tokens, or component variants | Recommend only when all four roles already exist. Show every value and source. Never generate or guess a missing role |
| `frustrationDetection` | no project signal required | Keep enabled (default `true`). Explain what it detects |
| `entryMode` | host-driven trigger buttons, no visible feedback tab expected | Only recommend `'api'` when the host already has a custom trigger |
| `userToken` / `userTokenFn` | Devise, NextAuth, Clerk, session middleware | Recommend `/makethisbetter setup-auth` for identity verification |
| `user` | unsigned user metadata in session | Only recommend when the project already exposes user info client-side |

Present **only the new options** in a single confirmation question. For each:
- State the option name, its SDK default, and the inferred recommendation with
  evidence.
- If a new option's SDK default is already the right choice for the project
  (e.g. `frustrationDetection: true` and no reason to disable), list it as
  "keeping SDK default" and do not write it into the `init()` call.

The user can accept, reject individual options, or provide explicit values.

### Color option rules

Same as `setup.md`:
- An invalid `brandColors` group falls back to SDK defaults.
- When recommending `brandColors`, show all four values and their codebase
  source. Never generate, darken, lighten, or guess a missing role.

## 5. Check Widget Version

Check the installed widget version against npm latest:

For Import Map / vendored installs:
```bash
# Check current vendor file for version hint
head -5 vendor/javascript/makethisbetter.js 2>/dev/null
# Check npm latest
npm view makethisbetter version
```

For npm installs:
```bash
# Check installed version
node -p "require('makethisbetter/package.json').version" 2>/dev/null || \
  jq -r '.dependencies.makethisbetter // .devDependencies.makethisbetter' package.json
# Check npm latest
npm view makethisbetter version
```

For CDN installs, check that the script tag uses `@1` (major-pinned):
```bash
grep -n "unpkg.com/makethisbetter" app/views/layouts/*.erb app/views/layouts/*.html *.html 2>/dev/null
```

If the installed version is behind npm latest, report it and suggest updating.
For Import Map projects, the update command is:

```bash
curl -sL "https://unpkg.com/makethisbetter@<VERSION>/dist/makethisbetter.standalone.js" -o vendor/javascript/makethisbetter.js
head -1 vendor/javascript/makethisbetter.js | grep -qE '^const |^var |^import |^/[/*]' || { echo "❌ Downloaded content is not JS, CDN may not have synced"; exit 1; }
```

For npm installs: `npm update makethisbetter`.
CDN `@1` installs auto-resolve to latest minor and need no update.

## 6. Apply Changes

Edit the `init()` call in-place, adding confirmed new options. Preserve:
- The existing `projectKey` and all currently configured options
- Code style (indentation, quotes, trailing commas)
- Surrounding code and imports

Write only non-default values. Example diff:

```diff
 MakeThisBetter.init({
   projectKey: "mtb_proj_xxx",
   locale: "zh-CN",
+  frustrationDetection: true,
+  brandColors: {
+    primary: "#2563eb",
+    hover: "#1d4ed8",
+    active: "#1e40af",
+    onPrimary: "#ffffff",
+  },
 })
```

## 7. Verify

1. The `init()` call is syntactically valid.
2. No duplicate keys exist.
3. When a dev server and browser tooling are available, load the page and check
   for widget console errors.

Finish with: file changed, options added, options kept at SDK default (not
written), version status, and any remaining recommendations (e.g.
`/makethisbetter setup-auth` for identity verification).
