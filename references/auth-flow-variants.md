# Auth Flow Variants

Most MCP setup pain lives in auth. Pick the right flow for your service. The flow you pick determines your reference doc and your `configure.sh`.

## Variant 1: OAuth 2.0 Desktop Client (most common)

The service is owned by a big provider (Google, Microsoft, Atlassian) and uses an OAuth flow with a desktop client.

**Examples:** Google Ads, Google Sheets, GA4, Gmail, Calendar, Drive, Atlassian (Jira, Confluence)

**The flow:**

1. Operator creates an OAuth 2.0 desktop client in the provider's developer console
2. Provider returns a client_id + client_secret
3. Operator runs a CLI tool (`google-oauthlib-tool` or similar) that opens a browser
4. Operator logs in, approves scopes, browser redirects to localhost
5. CLI tool captures the auth code, exchanges it for a refresh token
6. The refresh token gets written to the config file
7. The MCP server uses the refresh token to mint short-lived access tokens at runtime

**Setup skill needs:**
- A `references/oauth-desktop-client.md` walking through console creation (longest doc; this is where operators get stuck)
- A `scripts/configure.sh` that accepts the resulting refresh_token + client_id + client_secret as env vars and writes them to the credential file
- A verify step that confirms the refresh token works

**Where it goes wrong:**
- Operator picks Internal user type when they should pick External (or vice versa)
- Operator forgets to enable the API in the GCP project
- Operator skips the consent screen scopes step
- Operator's refresh token gets revoked after 7 days because the OAuth client is in Testing mode

Address each of these in the reference doc and in the troubleshooting section.

## Variant 2: OAuth 2.0 Web Flow

The service uses an OAuth flow that requires a callback URL on a real domain (not localhost).

**Examples:** Slack (legacy), Notion, some custom enterprise SSO

**The flow:**

1. Operator creates an OAuth app in the service's developer console
2. Operator specifies a callback URL (must be HTTPS, usually a real domain)
3. The MCP project provides a hosted callback handler OR the operator runs ngrok
4. Auth flow returns a long-lived bot/user token
5. Token gets written to the config file

**Setup skill needs:**
- A reference doc explaining the callback URL setup (this is the new pain)
- Optionally, a "use the hosted callback" path if the MCP project provides one
- A note about ngrok for self-hosted setups

**Where it goes wrong:**
- HTTPS required and operator only has HTTP
- Callback URL mismatch between OAuth app and the actual handler
- Operator's domain has CSP that blocks the redirect

## Variant 3: API Key (simplest)

The service exposes a static API key. Generate, copy, paste into config.

**Examples:** OpenAI, Anthropic, most paid SaaS APIs, Reoon, Hunter, RocketReach, Stripe

**The flow:**

1. Operator logs into the service
2. Navigates to API settings
3. Clicks "Create API Key"
4. Copies the key
5. Pastes into the config file (or env var)

**Setup skill needs:**
- A reference doc that's mostly a screenshot path ("click here, then here, then here")
- A `configure.sh` that takes the API key as an env var and writes it
- A note that the key is sensitive — never commit to git, store in a secrets manager for production

**Where it goes wrong:**
- Operator uses a read-only key when the MCP needs write access (or vice versa)
- API key expires (some services) or is revoked because of suspicious activity
- Operator has multiple environments (dev/prod) and uses the wrong key

## Variant 4: Service Account (cloud-platform pattern)

The MCP authenticates as a "service account" — a non-human identity managed by the cloud provider.

**Examples:** Google Cloud (BigQuery, Cloud Storage, Vertex AI), AWS (S3, IAM-roled services), Azure

**The flow:**

1. Operator creates a service account in the cloud provider's IAM
2. Grants the service account specific permissions
3. Downloads a JSON key file
4. Stores the JSON path in an env var (`GOOGLE_APPLICATION_CREDENTIALS`) or config
5. The MCP server authenticates using the JSON key

**Setup skill needs:**
- A reference doc on IAM permissions (which ones the MCP actually needs)
- A note that the JSON key is sensitive — gitignore aggressively
- A `configure.sh` that takes the JSON path as an env var

**Where it goes wrong:**
- Wrong scopes granted (too few, MCP can't do its job; too many, security risk)
- JSON file accidentally committed to git
- Service account hits per-project quota or rate limit
- IAM changes break the auth silently

## Variant 5: Personal Access Token (PAT)

A long-lived token tied to a human user. Common in developer-tool services.

**Examples:** GitHub, GitLab, Linear, Jira Cloud (newer), Notion (newer integration tokens)

**The flow:**

1. Operator logs into the service as themselves
2. Navigates to personal settings → developer/API → personal access tokens
3. Creates a new token with the required scopes
4. Copies the token (one-time display)
5. Pastes into the config

**Setup skill needs:**
- Reference doc with the scopes the MCP needs (this is the easy-to-miss part)
- A note that the token represents the operator personally — if they leave the company, the MCP breaks
- A `configure.sh` taking the token via env var

**Where it goes wrong:**
- Wrong scopes selected at token creation (need to make a new token)
- Token expires (some services force expiry; default 90 days on GitHub fine-grained)
- Token tied to a user who's no longer at the org

## How to pick

| The service is... | Use |
|---|---|
| Google / Microsoft / Atlassian | OAuth Desktop |
| A SaaS with a developer portal and OAuth | OAuth Web Flow |
| A SaaS with simple API keys | API Key |
| AWS / GCP / Azure | Service Account |
| GitHub / GitLab / Linear / dev-tool SaaS | Personal Access Token |

Most MCP servers document which they use. Read their README first. If unclear, look at the example config in their repo.

## What to put in the setup skill

For each variant, your `references/auth-setup.md` (or equivalent) should cover:

1. The exact console URL where the credential is created
2. Each click/dropdown/checkbox in order, with screenshots if possible
3. Exactly which scopes / permissions to grant — be specific, not generous
4. Where the resulting credential lives (JSON download, copy-paste, etc.)
5. How long the credential lasts and what to do when it expires
6. What goes wrong (per the variant's "where it goes wrong" list above)

Setup skills that skip the "scopes" detail get bug reports forever.
