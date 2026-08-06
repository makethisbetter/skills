# Identity Verification Setup

Run this workflow for `/makethisbetter setup-auth`.

## Security Boundary

- The Signing Secret lives only in the host application's server-side secret
  manager. Never put it in source code, browser JavaScript, HTML, logs, chat, or
  a client-visible environment variable.
- Do not ask an admin to paste or share the secret in chat. An account admin
  must store it directly in the target application's secret manager.
- JWTs must include `sub` and a short `exp`. Include `email` and `name` when the
  application has them; the server trusts the signed claims and does not fill
  missing claims from unsigned widget `user` data.
- Prefer `userTokenFn` so the Widget fetches a fresh short-lived JWT before each
  request. Static `userToken` is a supported fallback for a short-lived,
  server-rendered page.
- Never submit test Feedback to verify identity. It pollutes project data and
  consumes AI triage; validate the endpoint and request headers without finalizing
  a report.

## 1. Verify Access and Resolve the Project

Use a real authenticated request; `makethisbetter info` only shows local state
and returns success when no token is configured:

```bash
makethisbetter project list --json
```

If authentication fails, run `/makethisbetter setup`. Resolve the handle from
the single project or by matching the codebase domain. Ask only if multiple
projects remain plausible.

An account admin can read the project's `signing_secret`, but do not print the
full project response or the extracted value. First choose a concrete,
documented command that writes stdin into the target application's server-side
secret store without echoing it. Only then construct one pipeline with these
three stages: `makethisbetter project show <handle> --json`, `jq -er
'.signing_secret // error("Account admin access required")'`, and that concrete
secret-manager command. Enable `set -o pipefail` in the same shell, and require
the final writer to reject empty stdin. Never execute a placeholder or run the
first two stages by themselves. Afterward, use the secret manager's metadata or
status command to confirm the key exists without reading its value.

Do not use command substitution, a temporary plaintext file, or a command-line
argument that would expose the secret through process listings. If the target
secret manager has no non-printing write path, stop and have an account admin
configure it directly. If the field is absent, the current user is not an
account admin; do not work around that permission boundary.

## 2. Add an Authenticated Token Endpoint

Detect the backend and its existing session authentication, then add the
matching complete endpoint. Preserve the application's authentication stack;
do not introduce a second login mechanism. Every endpoint must reject a
signed-out request, return `{ "token": "..." }`, and set `Cache-Control:
no-store`.

### Rails

```ruby
# Gemfile
gem "jwt"

# config/routes.rb
namespace :api do
  get "makethisbetter/token", to: "make_this_better_tokens#show"
end

# app/controllers/api/make_this_better_tokens_controller.rb
class Api::MakeThisBetterTokensController < ApplicationController
  before_action :authenticate_user!

  def show
    payload = {sub: current_user.id.to_s, exp: 1.hour.from_now.to_i}
    payload[:email] = current_user.email if current_user.respond_to?(:email) && current_user.email.present?
    payload[:name] = current_user.name if current_user.respond_to?(:name) && current_user.name.present?

    signing_secret = Rails.application.credentials.dig(:makethisbetter, :signing_secret)
    raise KeyError, "MakeThisBetter signing secret is not configured" if signing_secret.blank?

    response.headers["Cache-Control"] = "no-store"
    render json: {token: JWT.encode(payload, signing_secret, "HS256")}
  end
end
```

### Express

```js
// npm i jsonwebtoken
const jwt = require("jsonwebtoken")

function requireAuthenticatedUser(req, res, next) {
  if (!req.user) return res.sendStatus(401)
  next()
}

app.get("/api/makethisbetter/token", requireAuthenticatedUser, (req, res) => {
  const signingSecret = process.env.MAKETHISBETTER_SIGNING_SECRET
  if (!signingSecret) return res.sendStatus(500)

  const payload = {
    sub: String(req.user.id),
    ...(req.user.email ? { email: req.user.email } : {}),
    ...(req.user.name ? { name: req.user.name } : {}),
  }
  const token = jwt.sign(payload, signingSecret, {
    algorithm: "HS256",
    expiresIn: "1h",
  })

  res.set("Cache-Control", "no-store").json({ token })
})
```

### Next.js App Router with NextAuth

```ts
// npm i jsonwebtoken
// app/api/makethisbetter/token/route.ts
import jwt from "jsonwebtoken"
import { auth } from "@/auth"

export async function GET() {
  const session = await auth()
  if (!session?.user?.id) return new Response(null, { status: 401 })

  const signingSecret = process.env.MAKETHISBETTER_SIGNING_SECRET
  if (!signingSecret) return new Response(null, { status: 500 })

  const payload = {
    sub: String(session.user.id),
    ...(session.user.email ? { email: session.user.email } : {}),
    ...(session.user.name ? { name: session.user.name } : {}),
  }
  const token = jwt.sign(payload, signingSecret, {
    algorithm: "HS256",
    expiresIn: "1h",
  })

  return Response.json({ token }, { headers: { "Cache-Control": "no-store" } })
}
```

### Python with Flask-Login

```python
# pip install pyjwt
from datetime import datetime, timedelta, timezone
from flask import current_app, jsonify
from flask_login import current_user, login_required
import jwt

@app.get("/api/makethisbetter/token")
@login_required
def makethisbetter_token():
    payload = {
        "sub": str(current_user.get_id()),
        "exp": datetime.now(timezone.utc) + timedelta(hours=1),
    }
    if getattr(current_user, "email", None):
        payload["email"] = current_user.email
    if getattr(current_user, "name", None):
        payload["name"] = current_user.name

    token = jwt.encode(
        payload,
        current_app.config["MAKETHISBETTER_SIGNING_SECRET"],
        algorithm="HS256",
    )
    response = jsonify(token=token)
    response.headers["Cache-Control"] = "no-store"
    return response
```

Adapt the authentication hook and optional `email`/`name` accessors to the
application's existing user model; never invent a value. Keep `sub` stable for
the same host-site account. `expiresIn` in the Node examples writes the required
`exp` claim. The endpoint returns only the signed JWT, never the Signing Secret.

## 3. Initialize the Widget

The browser integration should request a fresh token:

```js
MakeThisBetter.init({
  projectKey: "mtb_proj_xxx",
  userTokenFn: async () => {
    const response = await fetch("/api/makethisbetter/token", {
      credentials: "same-origin",
      headers: { Accept: "application/json" },
    })

    if (!response.ok) throw new Error("Unable to load MakeThisBetter user token")
    const { token } = await response.json()
    return token
  },
})
```

Use a static token only when the page is server-rendered and the page session
cannot outlive the token's validity window:

```js
MakeThisBetter.init({
  projectKey: "mtb_proj_xxx",
  userToken: token,
})
```

## 4. Verify

1. In development or test, call the endpoint as a signed-in user, decode the JWT
   locally, and confirm `sub`, `exp`, and available `email`/`name`; never log the
   Signing Secret.
2. Confirm the token endpoint rejects signed-out requests and returns
   `Cache-Control: no-store` for signed-in requests.
3. Initialize the Widget locally and inspect or intercept its next request to
   confirm `X-User-Token` is present. Stop before submitting/finalizing Feedback.
4. Confirm an expired token is rejected and `userTokenFn` obtains a replacement,
   using a request test or local interception rather than a real Feedback.
