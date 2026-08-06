# Setup

Run this workflow for `/makethisbetter setup`. Stop and report the exact failed
step if any command fails.

## 0. Read the Product README

Fetch and read the Make This Better skills repository README before proceeding:

```bash
curl -fsSL https://raw.githubusercontent.com/makethisbetter/skills/main/README.md
```

It contains the full commands reference, example session, and installation
guide. This context is required to make informed decisions during setup.

## Hard Rules

- Never read or print `~/.makethisbetter/config.json`; it contains a bearer
  token. Use `makethisbetter info` for a masked local status and an authenticated
  API command to verify the token.
- Infer options from the codebase before asking. Ask at most two questions total:
  one project-selection/name question when needed, then one configuration
  confirmation.
- When a complete semantic color group can be identified, the configuration
  confirmation must ask whether to use those exact colors across the complete
  Widget. Do not silently apply detected colors or invent missing colors.
- Keep the Signing Secret out of source code, browser code, chat, summaries, and
  logs. Normal widget setup needs the public Project Key, not the Signing Secret.
- Never submit test Feedback. It pollutes project data and consumes AI triage.

## 1. Ensure the CLI Is Available

Check every supported installation channel through the binary, not npm metadata:

```bash
command -v makethisbetter
```

If it is absent, use the primary installer:

```bash
npm install -g @makethisbetter/cli
```

Run `command -v makethisbetter` again. If npm failed on macOS or Linux, use the
prebuilt GitHub Release in a user-writable bin directory. Do not write directly
to `/usr/local/bin` and do not use `sudo`:

```bash
set -e
mtb_platform="$(uname -s | tr '[:upper:]' '[:lower:]')"
mtb_arch="$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')"
mtb_install_dir="${XDG_BIN_HOME:-$HOME/.local/bin}"
mtb_tmp_bin="$(mktemp "${TMPDIR:-/tmp}/makethisbetter.XXXXXX")"
mkdir -p "$mtb_install_dir"
trap 'rm -f "$mtb_tmp_bin"' EXIT
curl -fsSL "https://github.com/makethisbetter/cli/releases/latest/download/makethisbetter-${mtb_platform}-${mtb_arch}" -o "$mtb_tmp_bin"
test -s "$mtb_tmp_bin"
install -m 0755 "$mtb_tmp_bin" "$mtb_install_dir/makethisbetter"
```

If `$mtb_install_dir` is not already on `PATH`, stop and tell the user the exact
directory to add. On Windows, stop after an npm failure and report it; the shell
fallback above is only for macOS and Linux.

## 2. Verify Authentication Without Exposing the Token

Show the local configuration with its token masked:

```bash
makethisbetter info
```

`info` exits successfully even when no token is set and does not contact the
API. Verify the credential with an authenticated request:

```bash
makethisbetter project list --json
```

If the output says the token is missing, invalid, or expired, run:

```bash
makethisbetter login
```

Then repeat `project list --json`. Never infer a valid login from the `info`
exit code alone.

## 3. Detect the Framework and Installation Method

Use this first-match order:

| Signals | Installation |
|---|---|
| `config/importmap.rb` | Rails Import Map: `bin/importmap pin makethisbetter --from unpkg`, boot via meta tag + `turbo:load` in `application.js`, and add the shared widget partial below |
| `Gemfile` contains `rails` and `package.json` exists | Rails bundler: `npm i makethisbetter`, boot via meta tag + `turbo:load` in `application.js`, and add the shared widget partial below |
| `package.json` contains `next` | `npm i makethisbetter`; initialize from `layout.tsx` or `_app.tsx` in `useEffect` |
| `package.json` contains `nuxt` | `npm i makethisbetter`; initialize in a client-side Nuxt plugin |
| `package.json` contains `svelte` | `npm i makethisbetter`; initialize in `+layout.svelte` with `onMount` |
| `package.json` contains `vue` | `npm i makethisbetter`; initialize in `App.vue` with `onMounted` |
| `package.json` contains `react` | `npm i makethisbetter`; initialize in the root component with `useEffect` |
| `composer.json` or `wp-content/` | CDN script in `header.php` |
| `theme.liquid` or other `*.liquid` | CDN script in `theme.liquid` before `</head>` |
| `*.jsp` or `pom.xml` | CDN script in the shared page layout |
| `package.json` contains `react-native`, or `pubspec.yaml` | Unsupported: React Native and Flutter have no DOM. Explain this and stop |
| Anything else | CDN script in the main HTML layout |

Capacitor, Ionic, Cordova, and Tauri run in a webview and use the normal web
installation. React Native and Flutter must not fall through to the CDN path.

Read the project README and the relevant framework configuration before
choosing an edit location.

## Safe Project Output

For an account admin, project create/show/update responses include the Signing
Secret even though normal setup does not need it. Never run those commands with
their full response going to the terminal. Pipe JSON through an available
structured parser and keep only setup-safe fields. With `jq`:

```bash
jq -e '{id, handle, name, domain, api_key, board_url, ai_context}
  | select((.api_key | type) == "string" and (.api_key | length) > 0)'
```

Check that the parser exists before making the API call. If `jq` is unavailable,
use the codebase's installed JSON runtime to produce the same allowlisted shape;
do not run the command unfiltered first. Enable `pipefail` for every filtered
CLI command so an authentication, network, or API failure cannot be hidden by a
successful parser exit.

## 4. Resolve the Project

Reuse the successful `project list --json` response from authentication:

- Exactly one project: use it and name it in the final summary.
- Multiple projects: match `domain` against the repository name, package
  homepage, configured site URL, and git remote. If no confident match remains,
  ask which project to use and recommend the best candidate with evidence.
- No projects: infer a project name and bare domain from the repository. Ask for
  the name only, recommend the repository/site name, then create it:

```bash
set -o pipefail
makethisbetter project create "My App" --handle my-app --domain example.com --json \
  | jq -e '{id, handle, name, domain, api_key, board_url, ai_context}
    | select((.api_key | type) == "string" and (.api_key | length) > 0)'
```

Fetch the selected project's current configuration:

```bash
set -o pipefail
makethisbetter project show <handle> --json \
  | jq -e '{id, handle, name, domain, api_key, board_url, ai_context}
    | select((.api_key | type) == "string" and (.api_key | length) > 0)'
```

Use `api_key` as the widget `projectKey`. Do not send the user to the dashboard
for it.

## 5. Infer and Confirm Widget Configuration

Infer every option first. Present all inferred values and the AI Context draft
in one confirmation question, with evidence for every recommendation.

| Option | Signals | Recommendation |
|---|---|---|
| `locale` | `config/locales/`, i18n libraries, `<html lang>`, README language | Single language: set that locale. Multiple languages: interpolate the server/client locale |
| `theme` | theme toggle, Tailwind `dark:` classes, `prefers-color-scheme` | Dark mode support: keep default `auto`. Light-only: set `light` |
| `position` | Intercom, Crisp, Chatwoot, Drift, or another bottom-right control | Conflict: `left`. Otherwise keep default `right` |
| `tabText` | established product wording | Keep the localized default unless the product has a clearly different label |
| `brandColors` | An explicit semantic `primary` / `hover` / `active` / `onPrimary` group in CSS custom properties, Tailwind theme colors, design tokens, or component variants | Recommend the group only when all four roles already exist and each can be normalized to six-digit hex. Show every value and source. Never generate, darken, lighten, or otherwise guess a missing role |
| `frustrationDetection` | no project signal required | Keep enabled. Explain that it prompts after rage clicks, rapid navigation, form failures, or error pages |
| feedback board entry | existing global navigation, help/account menu, footer, and product terminology | Add one discoverable link to `board_url`. Reuse established wording; otherwise use `Feedback` in English or `反馈` in Simplified Chinese. Use `产品反馈` only when the shorter label is ambiguous. Prefer the help/account menu, then global navigation, then the footer |
| mobile reporter entry | touch-oriented or responsive UI | Keep the SDK default: no docked Feedback button on touch devices. Explain that frustration detection remains active. If the user wants a visible mobile entry, the host application must render its own button or menu item and call `MakeThisBetter.open()` |
| reporter identity | Devise, NextAuth, Clerk, session middleware | Logged-in users: recommend `/makethisbetter setup-auth`; do not rely on unsigned `user` data for verified identity |
| `apiUrl` | explicit self-hosted backend | Set only when the codebase already uses a self-hosted MakeThisBetter API |

The confirmation must name the proposed feedback board entry location and label.
When a complete semantic group was found, it must ask whether to use those exact
four colors across the complete Widget and show the value and codebase source for
each role. Do not calculate or assess a color scale. The user can accept the
recommendation, keep the Make This Better defaults, or provide explicit values.
Do not silently apply detected colors. An invalid `brandColors` group falls back
to the default complete Widget colors.

It must also state that touch devices keep automatic frustration detection but
have no SDK-provided Feedback button, then ask whether to preserve that default
or add a host-owned mobile button or menu item that calls `MakeThisBetter.open()`.
This remains part of the single configuration confirmation, not a third
question.

Write only non-default values:

```js
MakeThisBetter.init({
  projectKey: "mtb_proj_xxx",
  locale: "zh-CN",
  position: "left",
  brandColors: {
    primary: "#2563eb",
    hover: "#1d4ed8",
    active: "#1e40af",
    onPrimary: "#ffffff",
  },
})
```

## 6. Draft Project AI Context

Read the current `ai_context` from `project show --json`.

- If it is non-empty, leave it untouched and mention where it can be updated.
- If it is empty, draft 3-6 plain sentences from the README and codebase:
  what the product is, who uses it, the stack and key flows, and domain terms
  AI triage might otherwise misread.
- Use the product's primary language and no Markdown headings.
- Include the draft in the single configuration confirmation question.

After confirmation, write it through the CLI. MCP has no `project_update` tool:

```bash
set -o pipefail
makethisbetter project update <handle> --ai-context "<confirmed text>" --json \
  | jq -e '{id, handle, name, domain, api_key, board_url, ai_context}
    | select((.api_key | type) == "string" and (.api_key | length) > 0)'
```

## 7. Install the Widget

Apply the selected method and the confirmed `init()` options. Also add or reuse
one site-wide feedback board entry that links to the selected project's
`board_url`. Follow the host application's existing navigation and link patterns;
do not invent a standalone floating control for the board link. When there is no
clear primary location, prefer an existing help/account menu, then global
navigation, then the footer. Use the product's established label when it has one;
otherwise default to `Feedback` in English or `反馈` in Simplified Chinese. Use
`产品反馈` only when the shorter Chinese label is ambiguous in its placement.

On touch devices, the SDK deliberately provides no Feedback button while keeping
frustration detection active. Do not call `MakeThisBetter.showLauncher()` to add
one. Preserve the no-button default unless the user explicitly asks for a visible
mobile reporter entry.

When the user opts in, implement the entry in the host application's own mobile
UI using its existing button or menu-item component, then open the reporter from
that control:

```js
document.querySelector("#mobile-feedback")?.addEventListener("click", () => {
  MakeThisBetter.open()
})
```

The site-wide feedback board entry is still required when a host-owned mobile
reporter entry is added. The board entry opens the public board; the host-owned
control opens the in-page reporter.

For Rails Import Map, the command is required: it both writes the pin and
downloads the ESM package to `vendor/javascript`. Adding a bare line to
`config/importmap.rb` is incomplete.

```bash
bin/importmap pin makethisbetter --from unpkg
```

For CDN installs:

```html
<script src="https://unpkg.com/makethisbetter@1/dist/makethisbetter.js"></script>
```

### Turbo / Hotwire

Turbo replaces `<body>` during navigation. The widget needs two elements:

1. A **permanent host** div inside `<body>` so an open report or Interaction
   Replay survives Turbo page swaps.
2. A **config meta tag** that carries the project key, locale, colors, and
   current user identity. Turbo swaps this tag on each navigation, so the
   widget can detect login/logout changes and re-initialize.

Create a shared partial (e.g. `app/views/shared/_makethisbetter_widget.html.erb`):

```erb
<div id="mtb-widget-host" data-turbo-permanent></div>
<meta name="makethisbetter-config"
      data-project-key="<%= Rails.application.credentials.mtb_project_key %>"
      data-locale="<%= I18n.locale %>"
      <% if current_user %>
        data-user-id="<%= current_user.id %>"
        data-user-name="<%= current_user.name %>"
        data-user-email="<%= current_user.email %>"
      <% end %>>
```

Store the project key in Rails credentials (`bin/rails credentials:edit`), not
as a hardcoded string. Add `brandColors` data attributes when the project has
a semantic color group (see Step 5).

Render the partial in every layout:

```erb
<body>
  <%= yield %>
  <%= render "shared/makethisbetter_widget" %>
</body>
```

In `application.js`, boot the widget on each `turbo:load` and re-init when
the identity changes (login, logout, or locale switch):

```js
import { MakeThisBetter } from "makethisbetter"

let widgetIdentity = null
function widgetIdentityOf(meta) {
  const { projectKey, locale, userId, userName, userEmail } = meta.dataset
  return JSON.stringify({ projectKey, locale, userId, userName, userEmail })
}
function bootWidget() {
  const meta = document.querySelector('meta[name="makethisbetter-config"]')
  if (!meta) {
    if (widgetIdentity) { MakeThisBetter.destroy(); widgetIdentity = null }
    return
  }
  const identity = widgetIdentityOf(meta)
  if (identity !== widgetIdentity) {
    if (widgetIdentity) MakeThisBetter.destroy()
    const config = {
      projectKey: meta.dataset.projectKey,
      locale: meta.dataset.locale,
    }
    if (meta.dataset.brandPrimary) {
      config.brandColors = {
        primary: meta.dataset.brandPrimary,
        hover: meta.dataset.brandHover,
        active: meta.dataset.brandActive,
        onPrimary: meta.dataset.brandOnPrimary,
      }
    }
    if (meta.dataset.userId) {
      config.user = { id: meta.dataset.userId, name: meta.dataset.userName, email: meta.dataset.userEmail }
    }
    MakeThisBetter.init(config)
    widgetIdentity = identity
  }
}
document.addEventListener("turbo:load", bootWidget)
```

Do not call `MakeThisBetter.setLocale()` separately — the `turbo:load`
handler reads the updated locale from the meta tag on every navigation.

The host div must be inside `<body>` and needs both the stable id and
`data-turbo-permanent`. React, Vue, and Svelte routers do not replace `<body>`,
so they do not need this host solely for navigation persistence.

## 8. Verify Without Creating Feedback

1. `curl -sI https://unpkg.com/makethisbetter@1` returns 2xx or 3xx.
2. Rails Import Map installs have both the `makethisbetter` pin and the
   downloaded file under `vendor/javascript`.
3. The rendered page contains the widget import/script and initialization.
4. The rendered site-wide feedback board entry points to the selected project's
   `board_url` and follows the host application's existing responsive navigation.
5. On a touch-sized viewport, the SDK renders no Feedback button. If the user
   requested a visible mobile reporter entry, the host-owned control follows the
   application's UI patterns and opens the reporter with `MakeThisBetter.open()`.
   Frustration detection remains enabled unless the user explicitly disabled it.
6. When a dev server and browser tooling are available, load the page and check
   for widget console errors.

Finish with the project, installation method, final non-secret options and the
evidence behind them, feedback board entry location and label, mobile reporter
entry behavior, AI Context action, and how to run `/makethisbetter list`.
