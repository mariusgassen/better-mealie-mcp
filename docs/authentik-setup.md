# Authentik + Better Mealie MCP — click-by-click setup

This guide makes the MCP HTTP endpoint accept tokens issued by **your**
Authentik instance, so only your Authentik users can call it. Anyone else gets
`401`.

How it fits together:

- The MCP server is a *resource server* (RFC 9728). It never shows a login
  page — it validates the `Authorization: Bearer <token>` on each request.
- Your **AI tool** (ChatGPT connector, Claude connector, …) is the OAuth
  *client*. It runs the login against Authentik and sends the resulting token
  to the server.
- The server discovers your Authentik automatically from
  `MCP_AUTH_ISSUER/.well-known/openid-configuration`, fetches Authentik's
  `jwks_uri`, and verifies the token's RS256 signature, expiry, `iss`, and
  (optionally) `aud` and scopes.

Prerequisites:

- Authentik running, and you can log in to its **Admin interface**.
- A public HTTPS URL where your AI tool can reach the MCP server
  (`MCP_PUBLIC_BASE_URL`).

---

## Part A — Click-by-click in the Authentik Admin interface

### A1. Pick a signing key

The server accepts **RS256** tokens only. Authentik signs JWTs *symmetrically*
with the provider's client secret unless a signing key is selected, so the
provider **must** have one.

You have two options:

- **Simplest**: reuse the key Authentik already made — `authentik
  Self-signed Certificate`. You only need to select it in A3.
  (Downside: it is valid for one year; after expiry pick/renew the key.)
- **Recommended for a long-lived setup**: create a dedicated RSA key:

  1. In the Admin interface, go to **System Management → Certificates & Keys**
     (it may be under **Cryptography** in older versions).
  2. Click **Create**.
  3. **Name**: `Mealie MCP signing key`.
  4. Leave the **common name** / certificate fields as-is and make sure the
     type is **RSA** (2048 bits).
  5. Click **Create** (or **OK**).

   Note the key's name — you'll select it in step A3.

### A2. Create the application + provider

1. In the Admin interface, open **Applications → Applications**.
2. Click **Create with provider** (older versions: **New Application**). This
   makes the application *and* its OAuth2/OIDC provider in one go.
3. On the **New application** page:
   - **Name**: `Mealie MCP` (visible name only).
   - **Slug**: `mealie` (or anything short). ⚠️ This slug becomes part of your
     issuer URL (`/application/o/<slug>/`) — remember it.
   - Click **Next** (or **Save**).
4. On **Provider type**, select **OAuth2/OIDC** and click **Next**.

### A3. Configure the provider

On the **Configure OAuth2/OpenID Provider** page, set:

- **Client Type**: `Confidential` *(default)*. Your tool will use the client
  id + secret. (Only switch to `Public`/PKCE if your AI tool explicitly
  requires public clients.)
- **Client ID**: leave the auto-generated value.
- **Client Secret**: leave auto-generated. You'll copy it in A4.
- **Redirect URIs / Redirect URLs**:
  - Add the callback URL your AI tool documents (ChatGPT gives you a specific
    redirect URI per connector; the tool's docs state it).
  - Leaving the field empty also works: Authentik records the first redirect
    the tool sends on first sign-in. A wildcard `http://127.0.0.1:*` is handy
    for local testing if your tool uses a loopback redirect.
- **Signing Key**: select the key from A1 (`authentik Self-signed Certificate`
  or your `Mealie MCP signing key`). **Do not leave it empty** — an empty
  signing key means HS256, which this server rejects.
- **Scopes**: keep the default (`authentik default OIDC Configuration`,
  includes `openid`). Only if your tool needs refresh tokens, also add the
  `offline_access` scope mapping.
- Click **Create Application** (older versions: **Save**/**Finish**).

### A4. Copy the client credentials

1. Still in **Applications → Applications**, the new app is listed. Click its
   name to open it.
2. Read the **Client ID** (link/copy it).
3. Click **Reveal**/**Copy** next to the **Client Secret** (it is shown once;
   treat it like a password).
4. Keep both — they go into your **AI tool**, not into the MCP server.

### A5. Note the issuer URL

On the same application page (or on the provider page), find the section with
the **OpenID Configuration** / **Issuer** link. In the default *per-provider*
issuer mode it looks like:

```
https://auth.example.com/application/o/mealie/
```

This exact URL (your Authentik host + `/application/o/` + your slug from A2 +
`/`) becomes `MCP_AUTH_ISSUER`.

Quick sanity check — in a browser this URL should return JSON that includes a
`jwks_uri`:

```
https://auth.example.com/application/o/mealie/.well-known/openid-configuration
```

---

## Part B — Configure and run the MCP server

Create/extend the server's `.env`:

```env
# Remote API access to Mealie
MEALIE_BASE_URL=https://mealie.example.com
MEALIE_API_TOKEN=<a Mealie API token>

# Only accept tokens your Authentik issued
MCP_AUTH_MODE=oidc
MCP_PUBLIC_BASE_URL=https://mcp.example.com      # how clients reach YOU
MCP_AUTH_ISSUER=https://auth.example.com/application/o/mealie/

# Optional hardening (see README):
# MCP_AUTH_AUDIENCE=<provider client id>   # require the token's "aud"
# MCP_AUTH_SCOPES=openid,profile           # require these scopes
```

- `MCP_AUTH_AUDIENCE`: set it only if you confirmed the `aud` value your AI
  tool's token actually carries. Wrong value ⇒ every request `401`.
- `MCP_AUTH_SCOPES`: leave unset unless you need it; if set, the tool must
  request those scopes.

Prefer `MCP_AUTH_MODE=both` instead if you also want an API key for scripts:

```env
MCP_AUTH_MODE=both
MCP_AUTH_TOKEN=openssl rand -hex 32
```

Run:

```bash
uv run better-mealie-mcp --http 8000
```

Verify:

```bash
# Advertises your Authentik as the identity provider:
curl https://mcp.example.com/.well-known/oauth-protected-resource/mcp
# -> {"authorization_servers":["https://auth.example.com/application/o/mealie/"],...}

# No token ⇒ rejected:
curl -i https://mcp.example.com/mcp   # -> 401
```

---

## Part C — Configure the AI tool

- **Server URL**: `https://mcp.example.com/mcp`.
- **Authentication**: choose **OAuth / sign-in**.
  - Tools that implement RFC 9728 discovery only need the server URL — they
    fetch your Authentik from `/.well-known/oauth-protected-resource` and run
    the login themselves.
  - Tools without discovery: enter the **Client ID** and **Client Secret**
    from A4 and the **Authorization URL** (issuer + `authorize`) manually.
- Sign in: a browser opens, you log in to Authentik, and Authentik redirects
  back. Every subsequent request to `/mcp` carries the token.

---

## Part D — Troubleshooting

- **`401` right after login**: the token you received isn't valid for this
  check. Decode it (jwt.io) and confirm:
  - `alg` is `RS256` — if it's `HS256`, the provider has **no signing key
    selected** (see A3).
  - `iss` equals `MCP_AUTH_ISSUER` exactly (only a trailing `/` is ignored).
  - `aud` contains your `MCP_AUTH_AUDIENCE`, **if** you set it — otherwise
    remove the var.
- **`401` with `MCP_AUTH_SCOPES` set**: the tool didn't request those scopes.
  Remove the restriction or request the scopes.
- **Token expires quickly**: Authentik's default access-token lifetime is
  minutes; the tool refreshes automatically. If it can't, add `offline_access`
  in A3.
- **`pool refresh` / issuer unreachable at startup is fine**: the server
  fetches discovery lazily on the first request and 401s until Authentik is
  reachable.