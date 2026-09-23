# AGENTS.md — Seedful Tool

This file is the single source of truth for AI coding agents (Claude Code, Cursor,
Windsurf, Copilot) building a **tool on Seedful**. Follow these rules exactly.

> You are the Bob Ross of teaching how to build data tools with AI. The person you
> are helping may have never written code, opened a terminal, or designed a page
> before. Explain what you are doing in plain language, and never make them touch
> anything they do not have to.

> **What a Seedful tool is:** a **static web app** (plain HTML/CSS/JS, or a
> framework that builds to static files) that reads company data through
> Seedful's governed API and renders it — a dashboard, a report, a lookup,
> a small internal viewer. It has **no backend of its own**. All data comes
> from Seedful's gateway, and it is **read-only, end to end**. If you find
> yourself wanting to write a server, a database, or a login page, stop — that
> is not how Seedful tools work (see **What NOT to Do**).

> **When the docs don't tell you how to do something:** if this file, the spec
> catalog (`GET /v1/specs`), or the OpenAPI document (`GET /v1/openapi.json`)
> don't explain how to get some piece of data, **do not guess and do not invent
> an endpoint**. The data you need either lives in an approved spec or it does
> not exist yet. Ask the tool's owner to have a spec drafted and approved (see
> **Getting New Data: the Spec is the Contract**). Guessing a table name or a
> query will simply be refused by the gateway.

---

## The One Big Idea

Seedful separates two things that normally get tangled together:

1. **What may be queried** — decided by a human, once, per data source. An
   autonomous agent introspects a database and drafts a declarative **spec** (a
   YAML menu of named queries); a person reviews and **approves** it. Approval is
   the only human gate in the whole system.
2. **What tools do with it** — automated and self-serve. Once a spec is approved,
   any number of tools can read through it, and deploying a tool is a one-click,
   user-initiated action. This is safe **because a deployed tool can never touch
   anything except approved, read-only, scoped queries.**

Your job as the agent is entirely in world #2. You **consume** approved specs.
You never write one, never approve one, never reach past one. Everything below
follows from that.

---

## First-Time Setup

Before writing any feature code, get a running gateway to talk to.

1. **Find the gateway URL** — you only need it for testing directly against
   the gateway while you build. This repository deliberately does not contain
   it (it is public; Seedful writes no instance address or person's email into
   it): see your Seedful console, or ask the tool owner. Wherever this file
   says `<your-gateway-url>`, use that value, and keep it in one place in your
   code (a `GATEWAY` constant or an env var), never scattered. The deployed
   page itself never needs it: `index.html` calls the relative
   `/app/it-service-desk/api/query/...` proxy (see "Running a query").

2. **Confirm it is up:**
   ```bash
   curl <your-gateway-url>/health
   # -> {"ok":true}
   ```
   If this fails, the Seedful stack is not running. In a Seedful checkout the
   gateway starts with `pnpm --filter @seedful/gateway dev` (it needs the dev
   Postgres and IdP running too — see that repo's README). **You cannot fix a
   down gateway from inside a tool** — tell the owner it is unreachable.

3. **List what you are allowed to query — do this before writing any code:**
   ```bash
   curl -H "x-seedful-user: you@dev" -H "x-seedful-roles: employee" \
     <your-gateway-url>/v1/specs
   ```
   This returns every approved spec you can see: its name, its source, the roles
   allowed, and — for each query — the exact parameters, their types, bounds,
   enums, and defaults. **This is your menu.** Treat it the way the Storesight
   agent treats an OpenAPI spec: read it first, build against it, never guess
   past it.

4. **Only then** start building what the user asked for.

---

## Tool Identity

Filled in at deploy time. Keep it current as the tool changes:

- **Tool name:** IT Service Desk
- **Slug:** it-service-desk
- **Specs & queries used:** it_tickets.list_tickets, it_tickets.mean_time_to_resolve_by_priority, it_tickets.tickets_by_status
- **Roles required to view:** _derived from the `access_policy` of the specs
  above — every viewer must satisfy all of them_
- **Owner:** see this tool's page in your Seedful console

The **Specs & queries used** line matters: at deploy time Seedful mints this
tool a token scoped to exactly that allowlist. Keep it honest and minimal — a
query you list but don't use just widens the token for no reason. Adding a
query here later does not widen the deployed token by itself — re-deploy (or
re-issue the token from the tool's page) once the query is actually used.

---

## Data Access

**All data comes from the Seedful gateway. There is no other source.** No direct
database connections, no connection strings, no second API. If a value isn't
reachable through an approved query, the tool does without it.

### Discover before you build

Two read-only endpoints describe everything on offer. Fetch them **before**
writing query calls — they are authoritative; this file is not.

```bash
# The full catalog: every spec, query, and parameter you may use.
curl <your-gateway-url>/v1/specs

# One spec in detail.
curl <your-gateway-url>/v1/specs/customers

# The same catalog as an OpenAPI 3 document, if you prefer that shape.
curl <your-gateway-url>/v1/openapi.json
```

A spec summary looks like this (the gateway **never** returns SQL — only the
callable surface):

```json
{
  "name": "customers",
  "source": "mockco-postgres",
  "roles_allowed": ["employee"],
  "queries": [
    {
      "name": "declining_usage",
      "description": "Customers whose usage dropped >30% over the trailing 90 days.",
      "params": [
        { "name": "limit", "type": "integer", "required": false,
          "default": 25, "max": 100 }
      ]
    }
  ]
}
```

Use the catalog to answer: which query gives me this data? what params does it
take? what are the type, bounds, and enum on each? what does it return? **Do not
hardcode assumptions about columns or shapes — read the catalog.**

### Running a query

One endpoint runs queries. You pick a **named** query from an approved spec and
pass **typed params** in the body. Freeform SQL is never accepted — there is
nowhere to put it.

```
POST /v1/query/{spec}/{query}
Content-Type: application/json

{ "params": { "limit": 25 } }
```

In the page Seedful serves, call the tool's own relative proxy path,
`/app/it-service-desk/api/query/{spec}/{query}` (the starter `index.html` already
does). Calling the gateway directly, from any JavaScript, is only for local
testing — see "Local Development" below:

```js
const GATEWAY = "<your-gateway-url>"; // from your Seedful console; never needed in the served page

async function runQuery(spec, query, params = {}) {
  const res = await fetch(`${GATEWAY}/v1/query/${spec}/${query}`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    // Dev identity headers go here for local testing — see "Supplying a dev
    // identity" below. Never ship them; the gateway rejects them outside dev.
    body: JSON.stringify({ params }),
  });
  if (!res.ok) {
    const { error } = await res.json();          // { code, message }
    throw new Error(`${error.code}: ${error.message}`);
  }
  return res.json();
}

// Example
const result = await runQuery("customers", "declining_usage", { limit: 25 });
// result = { spec, query, row_count, rows: [...], meta: { params } }
```

**The starter `index.html` in this repo does NOT call the gateway directly.**
Once Seedful is serving the tool (dev or deployed), it calls its own
same-origin data proxy instead — a relative `/api/query/{spec}/{query}`, path
relative to wherever the tool is served — because that is what actually
attaches the tool's scoped token: server-side, with nothing for this code to
hold. The `GATEWAY` form above is for testing directly against the real
gateway while you build; it is not what runs once the page is open through
Seedful.
renderChart(result.rows);
```

The response is always:

```json
{
  "spec": "customers",
  "query": "declining_usage",
  "row_count": 12,
  "rows": [ { "company": "...", "mrr_cents": 369200, "...": "..." } ],
  "meta": { "params": { "limit": 25 } }
}
```

### Rules for calling queries

- **Only names from the catalog.** A spec or query the gateway doesn't serve
  comes back `unknown_spec` / `unknown_query` (404). That is not a bug to work
  around — it means the query isn't approved.
- **Every param is validated against the spec** (type, min, max, enum, required)
  before any SQL runs. Send the right types; respect the bounds. A bad param is
  `invalid_params` (400).
- **`limit` is mandatory and capped.** Every query declares a `limit` with a
  maximum. You cannot page past it in one call — request a page, then page with
  the query's `offset` param if it has one.
- **Fetch sequentially, not in a storm.** Deployed tools are rate-limited per
  token (a `429` carries a `Retry-After` header). Ask for the data you need in a
  few calls, not dozens of parallel ones. Paginate rather than fan out.

### Error handling

Every refusal is a JSON body `{ "error": { "code": "...", "message": "..." } }`
with a matching HTTP status. Handle them; don't retry blindly.

| code | status | meaning | what to do |
|---|---|---|---|
| `unknown_spec` / `unknown_query` | 404 | not an approved, servable query | you guessed — go back to `/v1/specs` |
| `invalid_params` | 400 | a param failed type/bounds/enum/required | fix the param to match the spec |
| `unauthenticated` | 401 | no valid identity or token | dev: supply dev headers; prod: token issue, tell the owner |
| `forbidden` | 403 | identity/role or token scope not allowed | this viewer or tool may not run this query |
| `rate_limited` | 429 | too many calls | back off; honor `Retry-After` |
| `internal_error` | 500 | a gateway bug | surface a friendly message; don't loop |

```js
try {
  const result = await runQuery("customers", "declining_usage", { limit: 25 });
  render(result.rows);
} catch (e) {
  // Show an empty/error state — never a stack trace to the user.
  showMessage("Couldn't load churn data right now.");
  console.error(e);
}
```

---

## Getting New Data: the Spec is the Contract

If the data you need is not in any approved spec, **you cannot add it from the
tool.** There is no override, no direct query, no "just this once." This is the
whole security model, not an obstacle to route around.

The correct path:

1. Tell the tool's owner exactly what data you need and why (which fields, from
   which source, for what view).
2. The owner has Seedful's **agent** introspect the source and **draft** a spec
   (or extend an existing one) with the query you need.
3. A human **reviews and approves** the draft in your company's Seedful
   console.
4. Once approved, the query shows up in `GET /v1/specs` and you can build against
   it — no tool redeploy needed to *see* it, though you'll add it to the tool's
   allowlist when you use it.

Note that **sensitive fields are simply left off the spec** — they aren't hidden
behind a flag, they're absent. If a column you want isn't in the catalog, assume
it was left off deliberately. Ask; don't hunt for a back door (there isn't one).

---

## Identity, Tokens, and Who Sees What

You mostly don't handle any of this — Seedful does — but understand the model so
you don't try to reinvent it.

- **Deployed tools carry one scoped token.** When the owner deploys your tool,
  Seedful mints a single token scoped to the exact `spec.query` allowlist from
  your **Tool Identity**, with a rate limit and an expiry, revocable in one
  click. The token stays **server-side in Seedful's hosting layer** and is
  attached to the tool's queries for you. It is **never** sent to the browser
  and **never** written into your code.
- **The viewer is authenticated by SSO.** Anyone opening the deployed tool hits
  the company's single sign-on first. Their identity is attached to every query
  for the **audit log** (who ran what, when, through which tool).
- **Role gates the view.** A viewer must satisfy the `access_policy.roles_allowed`
  of *every* spec the tool uses. If they don't, Seedful shows a "request access"
  placeholder instead of the tool. You don't write this check — Seedful derives
  it from your query allowlist at deploy time.

So: **do not build a login page, do not store a token, do not add auth
middleware.** There is nowhere in a static tool for a secret to live safely, and
Seedful already owns this end to end.

### Publishing a new version: push, then Pull latest

Seedful serves the tool from **one pinned commit** of this repository, not from
whatever the default branch holds right now. So a `git push` on its own changes
nothing that viewers see. To publish:

1. Commit and push to the repository's **default branch**.
2. The tool owner (or an approver) opens the tool's page in the Seedful console
   and presses **Pull latest**, pasting a GitHub personal access token that is
   used once and never stored. Seedful then pins the new head commit.

Do not rename or transfer the repository, or rename the GitHub account that owns
it: Seedful pins the repository's GitHub id and refuses to serve ("Tool source
changed") if the name stops resolving to that exact repository. Tell the owner
to do this step when you finish a change — you cannot do it for them.

### How your tool authenticates to its data proxy

Seedful serves your page **sandboxed** (`Content-Security-Policy: sandbox
allow-scripts allow-forms`): it runs with an opaque origin and the browser sends
no Seedful cookie with anything it requests. Instead, each time the page is
served Seedful injects a short-lived **view grant** (10 minutes) as
`<meta name="seedful-view-grant" content="…">` and `window.SEEDFUL_VIEW_GRANT`,
and patches `fetch()` so relative calls to `/app/<your-slug>/api/query/…` carry
`Authorization: Bearer <grant>` automatically. The starter `runQuery` needs no
change. If you call the proxy some other way (XHR, a library), send that header
yourself. A `401` with `error.code: "grant_expired"` means the page has been
open too long — reload it.

While tools are served on the console's origin (until each company's dedicated
tool origin exists):

- **Keep the tool self-contained in `index.html`** — inline `<style>` and
  `<script>`, images as data URIs. Separate `.js`/`.css`/image files are
  requested without a cookie and will not load.
- **`localStorage`/`sessionStorage`/cookies are unavailable** in the sandbox;
  keep UI state in memory.
- No popups (`window.open`) and no framing.

---

## Local Development

A static tool is just files — open `index.html`, or serve the folder with any
static server. The only moving part is where it gets data.

### Point the tool at a running gateway

Set your `GATEWAY` constant to your gateway URL (see your Seedful console, or ask
the tool owner) and make sure the Seedful stack is running. Confirm with
`curl <your-gateway-url>/health`.

### Supplying a dev identity

In production the scoped token is attached for you. In local dev there is no
hosting layer, so the gateway accepts **dev identity headers** on a request that
has no `Authorization` header (this is on by default locally and **off** in any
real deployment):

```
x-seedful-user: you@dev
x-seedful-roles: employee
```

Use these while iterating — e.g. add them in your fetch during dev, or drive the
gateway from `curl`:

```bash
curl -X POST <your-gateway-url>/v1/query/customers/declining_usage \
  -H "Content-Type: application/json" \
  -H "x-seedful-user: you@dev" -H "x-seedful-roles: employee" \
  -d '{"params":{"limit":25}}'
```

**Do not ship the dev headers.** They exist only for local iteration; strip them
before deploy (a deployed gateway rejects them). A simple pattern: only add the
headers when `GATEWAY` points at `localhost` — the starter `index.html` does this.

### Heads-up: CORS in local dev

In production Seedful serves your tool **same-origin** with its gateway, so
`fetch()` from the page just works and the scoped token is attached server-side.
In local dev there is no hosting layer and the gateway sends no CORS headers, so
a page opened from disk (`file://`) or served on a different port **cannot fetch
the gateway directly** — the call fails and your tool lands on its error state.
That's expected, not a bug in your tool. For local work:

- **Verify queries with `curl`** (with the dev headers, as shown above) — this is
  the reliable way to confirm a query, its params, and its response shape locally.
- **Iterate on layout against a snapshot** — paste a few real rows into the page
  and build the visuals, then rely on same-origin serving in production for live
  data. The reference tool `demo/churn-risk-dashboard.html` works this way.

### Working snapshot vs. live data

While designing the layout it's fine to hardcode a small **snapshot** of real
rows (pulled once from the gateway) so you can iterate on visuals without the
stack running — the reference tool in this repo (`demo/churn-risk-dashboard.html`)
does exactly this. Just make it obvious in a comment that it's a snapshot, and
wire it to a live `runQuery` call before the tool is considered done.

---

## Building with an MCP Client (Claude Code / Cursor)

If you're exploring the data interactively from an MCP-capable editor rather than
writing fetch calls, the same gateway speaks **MCP over Streamable HTTP**. Each
approved query is exposed as a tool named `spec__query` (double underscore) — for
example `customers__declining_usage`. `tools/list` shows only the queries your
role may run; the params are generated from the spec, so the same type and bound
rules apply. This is the dev-time door for a human building a tool; the deployed
tool itself always uses the REST endpoints above.

---

## Style

- **Static and self-contained.** Plain HTML/CSS/JS is the default and usually
  enough. Reach for a framework only if the tool's complexity genuinely warrants
  it — and it must still build to static files with no server.
- **Read-only, always.** The tool displays data. It never claims to change
  anything, because it can't.
- **Design for the empty and error states.** A query can return zero rows, or
  fail. Show something calm and clear, never a blank page or a stack trace.
- **Accessible and legible.** This is an internal tool people rely on — real
  labels, keyboard focus, readable contrast in light and dark. The reference
  dashboard is a good bar to clear.
- **Keep it small.** Minimal dependencies, obvious data flow. Someone should be
  able to read the whole tool in one sitting.

---

## What NOT to Do

- Do **NOT** connect to any database directly (Postgres, Elasticsearch, anything)
  or use a connection string. All data goes through the gateway.
- Do **NOT** write or send SQL. You choose a **named query** and pass typed
  params — there is no place for freeform SQL and the gateway won't accept it.
- Do **NOT** guess spec names, query names, columns, or params. Read `/v1/specs`.
- Do **NOT** build a backend, an API, or server-side code. Seedful tools are
  static web apps.
- Do **NOT** build a login page or any auth mechanism, and do **NOT** store,
  embed, or log a token. SSO and the scoped token are Seedful's job.
- Do **NOT** try to write, update, or delete anything anywhere. Seedful is
  read-only end to end, at every layer.
- Do **NOT** persist tool data in some sideband store (a shadow database of
  records is a write path, which doesn't exist in v1). Browser storage is not
  available to a served tool at all — see "How your tool authenticates to its
  data proxy".
- Do **NOT** ship the dev identity headers, and do **NOT** try to raise your own
  rate limit — back off and paginate instead.
- Do **NOT** call external APIs or load remote scripts to fetch data. The tool's
  only data source is the Seedful gateway.
- Do **NOT** attempt to reach data that isn't in an approved spec. If you need
  it, ask the owner to have a spec drafted and approved — that is the only way,
  by design.
