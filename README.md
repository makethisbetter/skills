<p align="center">
  <img src="https://makethisbetter.dev/icon.svg" width="80" height="80" alt="Make This Better">
</p>

<h1 align="center">Make This Better Skills</h1>

<p align="center">
  Type <code>/makethisbetter list</code> and your agent knows what your users need.
</p>

<p align="center">
  <a href="https://makethisbetter.dev">makethisbetter.dev</a> &middot;
  <a href="https://github.com/makethisbetter/skills/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="license"></a>
  <a href="https://docs.anthropic.com/en/docs/claude-code/skills"><img src="https://img.shields.io/badge/Claude_Code-Skills-F97316" alt="Claude Code Skills"></a>
</p>

---

## What This Does

Skills give Claude Code slash commands for working with user feedback. Your users report bugs through the [widget](https://github.com/makethisbetter/makethisbetter-js). AI triages them. You type `/makethisbetter pick acme/FB-123` and your agent reads the full context — screenshot, console errors, AI analysis — and starts writing a fix.

This is how AI-native feedback works. No context-switching, no ticket grooming, no copy-pasting error messages. Your agent gets structured feedback and acts on it.

Skills orchestrate the [Make This Better CLI](https://github.com/makethisbetter/cli) under the hood. You don't need to know CLI syntax — the agent runs the right commands and reads the output.

## Install

```bash
npx skills add makethisbetter/skills
```

Auto-detects your installed agents (Claude Code, Cursor, Windsurf, etc.) and installs the skill to the right location. Powered by [`vercel-labs/skills`](https://github.com/vercel-labs/skills).

Or clone manually (the skill lives in the repo's `makethisbetter/` subdirectory):

```bash
git clone --depth 1 https://github.com/makethisbetter/skills.git /tmp/mtb-skills \
  && cp -R /tmp/mtb-skills/makethisbetter ~/.claude/skills/makethisbetter \
  && rm -rf /tmp/mtb-skills
```

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with Skills support

The [Make This Better CLI](https://github.com/makethisbetter/cli) is installed automatically when you run `/makethisbetter setup`. Or install it manually:

```bash
go install github.com/makethisbetter/cli@latest
```

## The Magic Moment

```
You: /makethisbetter pick acme/FB-1

Agent: Claimed acme/FB-1. Reading context...

  Page: /login
  Browser: Safari 18.2
  Console: TypeError: Cannot read property 'submit' of null
  Screenshot: [annotated screenshot showing the login form]
  AI analysis: Form submission handler not finding the form element
  in Safari due to DOM ready timing issue. The querySelector runs
  before the form is in the DOM on Safari 18.2.

  I see the issue — the login form handler binds on DOMContentLoaded
  but Safari 18.2 fires that event before deferred scripts finish
  parsing. Let me wrap the binding in a requestAnimationFrame...

  [agent edits app/javascript/controllers/login_controller.js]
  [agent runs bin/rails test test/system/login_test.rb — passes]

You: Looks good. /makethisbetter resolve acme/FB-1

Agent: Marked acme/FB-1 as shipped. Linked PR #42.
       The user who reported this will be notified that their fix is live.
```

That's the full loop: user reports a bug, AI triages it, your agent fixes it, the user hears back. All from your terminal.

## Commands Reference

All commands live under the `/makethisbetter` namespace:

### Setup & Authentication

```
/makethisbetter setup       Install CLI, authenticate, configure and install the widget
/makethisbetter login       Log in via email OTP
/makethisbetter signup      Create account (same as login — find-or-create)
/makethisbetter setup-auth  Set up identity verification (JWT signing)
```

`/makethisbetter setup` is the entry point — the whole flow runs in your terminal, no dashboard visit needed. It installs the CLI if needed, logs in via email OTP, resolves your project (listing or creating one through the API — the project key comes back in the response), then runs a guided widget configuration interview: the agent infers `locale`, `theme`, `position`, and identity options from your codebase, asks you at most two questions, and writes the final `MakeThisBetter.init({...})` into your code. Config is stored at `~/.makethisbetter/config.json`.

`/makethisbetter setup-auth` walks you through adding identity verification. It reads your project's signing secret via `makethisbetter project show` (account admins only), detects your backend framework (Rails, Express, Next.js, Python), and shows you exactly how to generate a JWT and pass it to `MakeThisBetter.init({ userToken: token })`. Feedback submitted with a valid token shows the verified user identity in the dashboard.

### Project Management

```
/makethisbetter setup       (includes project resolution)
```

Under the hood the agent uses the CLI directly:

```
makethisbetter project list              List projects with feedback counts
makethisbetter project show <handle>     Project detail incl. projectKey (api_key), board URL
makethisbetter project create <name> --handle <handle>     Create a project (account admins)
```

### Feedback Management

#### List feedback

```
/makethisbetter list
/makethisbetter list --status received
/makethisbetter list --label Safari
/makethisbetter list --priority critical
/makethisbetter list --sort priority
```

Shows a table of feedback items. Your agent can scan it and decide what to work on.

#### Pick and start working

```
/makethisbetter pick acme/FB-1
```

This is the power command. It:

1. Sets the feedback status to `in_progress`
2. Returns the full context: description, screenshot, console errors, AI triage analysis, page URL, browser, OS
3. Your agent reads all of this and starts making code changes

#### Dismiss

```
/makethisbetter dismiss acme/FB-1 --reason duplicate
```

Declines a feedback item. Sets status to `closed` with reason `not_planned`.

#### Resolve

```
/makethisbetter resolve acme/FB-1
/makethisbetter resolve acme/FB-1 --pr https://github.com/org/repo/pull/42
```

Marks feedback as shipped. Auto-detects the PR URL from the current git branch if `--pr` is not provided. The reporter gets notified that their fix is live.

## Example Session

A realistic Claude Code session — setup through resolution:

```
You: /makethisbetter setup
Agent: CLI installed. Logged in as you@company.com.
       No project yet — what should I name it? I'd suggest "Acme"
       (from your repo name).

You: Acme works.
Agent: Created project Acme (mtb_proj_x9f2...).
       Detected Rails 8 with import maps, config/locales has en+zh-CN,
       and Tailwind dark: classes. My recommended widget config:

         locale: '<%= I18n.locale %>'   (site is bilingual)
         theme: auto                     (dark mode supported — default)
         position: right                 (no competing widgets — default)
         frustrationDetection: on        (prompts after rage clicks — default)

       Apply this?

You: Yes.
Agent: Pinned "makethisbetter" in config/importmap.rb, added init to
       application.js. Verified: CDN reachable, init present on the
       rendered page. You never had to open the dashboard.

You: /makethisbetter list --status received --label Safari
Agent: Found 3 unresolved Safari feedback items:

  acme/FB-1  critical  "Login button doesn't work on Safari"
  acme/FB-2  high      "Dashboard chart shows wrong timezone"
  acme/FB-3  medium    "Typo in settings page header"

You: Start with the critical one.
Agent: /makethisbetter pick acme/FB-1
       Claimed acme/FB-1. Reading context...

       Page: /login
       Browser: Safari 18.2
       Console: TypeError: Cannot read property 'submit' of null
       AI analysis: Form submission handler not finding the form
       element in Safari due to DOM ready timing issue.

       I see the problem. The login_controller.js binds on
       DOMContentLoaded but Safari 18.2 fires that before deferred
       scripts finish. Let me fix this...

       [edits app/javascript/controllers/login_controller.js]
       [runs bin/rails test test/system/login_test.rb — 1 test, 0 failures]

       Fixed. The form binding now uses requestAnimationFrame as a
       fallback. Tests pass. Ready to resolve.

You: /makethisbetter resolve acme/FB-1
Agent: Marked acme/FB-1 as shipped. Linked PR #42.
       The reporter will be notified that the fix is live.

You: Nice. What's next?
Agent: acme/FB-2 — "Dashboard chart shows wrong timezone" (high priority).
       Want me to pick it up?
```

## Architecture

```
Claude Code
  ├── /makethisbetter setup      → runs: makethisbetter login
  │                                     + makethisbetter project list/create/show
  │                                     + guided widget configuration interview
  ├── /makethisbetter setup-auth → guides identity verification (JWT signing)
  │                                     via makethisbetter project show (signing secret)
  ├── /makethisbetter list       → runs: makethisbetter feedback list --project <handle> --json
  ├── /makethisbetter pick <handle/FB-n>  → runs: makethisbetter feedback show <handle/FB-n> --md
  │                                     + makethisbetter feedback pick <handle/FB-n>
  ├── /makethisbetter dismiss    → runs: makethisbetter feedback dismiss <handle/FB-n> --reason not_planned
  └── /makethisbetter resolve    → runs: makethisbetter feedback resolve <handle/FB-n> --pr <url>
        ↕
  Make This Better CLI (Go binary, installed locally)
        ↕
  Make This Better API (REST, https://makethisbetter.dev/api/v1)
        ↕
  Dashboard + AI Triage + User Notification
```

Skills orchestrate CLI commands. For direct MCP tool integration (no CLI needed), use the [MCP Server](https://github.com/makethisbetter/mcp) instead.

## Self-Hosting

Skills work with self-hosted Make This Better instances. Set `api_url` in `~/.makethisbetter/config.json` to point at your own server:

```json
{
  "api_token": "token_xxx",
  "api_url": "https://feedback.yourcompany.com/api/v1"
}
```

The platform backend is not open source yet — self-hosting docs will come with it. The hosted service lives at [makethisbetter.dev](https://makethisbetter.dev).

## Related

| Package | What it does | Install |
|---------|-------------|---------|
| [Make This Better](https://makethisbetter.dev) | The platform — dashboard, AI triage, feedback board | — |
| [Widget SDK](https://github.com/makethisbetter/makethisbetter-js) | Collect feedback from your website | `npm i makethisbetter` |
| [CLI](https://github.com/makethisbetter/cli) | Terminal commands (engine behind Skills) | `go install github.com/makethisbetter/cli@latest` |
| [MCP Server](https://github.com/makethisbetter/mcp) | Native AI agent integration via MCP | `npx -y @makethisbetter/mcp` |

## License

[MIT](LICENSE)
