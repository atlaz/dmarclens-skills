---
name: dmarclens
description: Read a DMARC Lens account through its read-only API or MCP server. Also covers its free DMARC, SPF, DKIM, ARC, MTA-STS, TLS-RPT and BIMI tools, which need no account. Use when the user asks about DMARC, SPF or DKIM status of their domains, who is sending email as them, pass rates, threat levels, DNS record health, alerts, or wants a state-of-play report for a company — and they use DMARC Lens (app.dmarclens.io or a self-hosted instance).
license: MIT
metadata:
  author: DMARC Lens
  homepage: https://www.dmarclens.io/
  version: "1.1"
---

# DMARC Lens

DMARC Lens collects DMARC aggregate and failure reports, names every sending
source, grades the threat to each domain, checks DMARC, SPF, DKIM, MX, MTA-STS,
TLS-RPT and BIMI records, and raises alerts. Domains belong to companies;
companies belong to one organisation. This skill reads that data. It cannot
change anything: the API and the MCP server are read-only by design.

## Two ways in

| Way | When to use it | Base |
|---|---|---|
| MCP server | The agent host supports MCP connectors (Claude, ChatGPT, Cursor, VS Code…). OAuth sign-in, no key handling. | `https://app.dmarclens.io/api/mcp` |
| REST API | Scripts, notebooks, hosts without MCP. One bearer key. | `https://app.dmarclens.io/api/v1` |

For a self-hosted instance replace `app.dmarclens.io` with its host. Both ways
see exactly what the signed-in person sees in the app: their organisation, and
all companies or only the ones they were given.

## Getting access

- **MCP:** add `https://app.dmarclens.io/api/mcp` as a custom connector / MCP
  server. It advertises OAuth 2.1 discovery at
  `/.well-known/oauth-protected-resource/api/mcp`, supports dynamic client
  registration and PKCE, and shows a consent page in the browser. No secret to
  paste.
- **API key:** in the app, open **API** (top navigation, `/api-access`), create
  a key with a name and optional expiry. Keys start with `dml_` and are shown
  once. Send them as `Authorization: Bearer dml_…`. Keys are personal; up to 10
  per person; 120 requests a minute. API and MCP access is part of the
  organisation's plan, so a 403 means the plan does not include it.

Never ask the user to paste a key into chat if the host can hold secrets
elsewhere (environment variable, connector settings). Never print a key back.

## Start by learning the names

Always call `list_companies` (MCP) or `GET /api/v1/companies` first. Company
and domain parameters accept the **id or the exact name**; the names come from
that call. Do not guess domain names from the user's wording.

## MCP tools

| Tool | What it returns | Arguments |
|---|---|---|
| `whoami` | The person, their role, organisation and visible companies | — |
| `list_companies` | Companies with their domains | — |
| `get_overview` | Totals, pass rate, daily series, one row per company | `days` (1–365, default 30) |
| `get_company` | Totals, series, a row per domain (policy, threat level, pass rate) | `company`, `days` |
| `get_company_insights` | Prioritised steps (now / next / later) for each domain of the company | `company` |
| `get_company_report` | State-of-play report in Markdown: summary, what is wrong, risks, remedies | `company` |
| `list_domains` | Every domain with DNS health and the issues found | — |
| `get_domain` | Traffic by sender and category, published policy, DNS checks, recent changes, progress towards `p=reject` | `domain`, `days` |
| `get_domain_insights` | Prioritised steps for one domain | `domain` |
| `list_alerts` | Open alerts (new unknown sender, pass-rate drop, DNS change, nameservers out of sync, reports stopped, threat level raised, flattened SPF out of date) | `include_closed` |

## REST endpoints

All under `/api/v1`, all `GET`, JSON unless noted. `GET /api/v1` itself needs no key and lists them.

```
/me
/overview?days=30
/companies
/companies/{id or name}?days=30
/companies/{id or name}/insights
/companies/{id or name}/report        (JSON)
/companies/{id or name}/report.md     (Markdown)
/domains
/domains/{id or name}?days=30
/domains/{id or name}/insights
/alerts?include_closed=false
```

Example:

```bash
curl -s -H "Authorization: Bearer $DMARCLENS_KEY" \
  "https://app.dmarclens.io/api/v1/domains/example.com?days=30"
```

## Reading the data correctly

- **Categories of senders:** `compliant` (passes DMARC), `forwarded` (a list or
  forwarder re-sent it; includes mail rescued by ARC), `misconfigured` (a known
  service of the owner that fails authentication: fix its setup), `unknown`
  (nobody has reviewed it), `threat` (an administrator marked it). Only an
  administrator assigns `threat`; never call an `unknown` sender an attacker.
- **Threat level** per domain, from the last 7 days: `low`, `elevated`
  (spoofing seen but all quarantined or rejected), `high` (spoofed mail
  delivered), `critical` (at volume, or 5% or more of the domain's mail).
- **Pass rate** is DMARC-aligned pass, not raw SPF or DKIM pass. `spf_rate` and
  `dkim_rate` are alignment rates too.
- **Policy progress** says whether a domain is ready for `quarantine` or
  `reject`; follow it rather than recommending `p=reject` on your own.
- **Volume limits** in a plan never drop reports; `over` only flags it.
- **Dates** are ISO 8601 in UTC; `days` windows end today.

## Typical tasks

- *"How are our domains doing?"* → `get_overview`, then `list_domains` for the
  health lights; summarise pass rate, threat levels, and the worst DNS issues.
- *"What should we fix first for Acme?"* → `get_company_insights("Acme")`; the
  `now` items are the answer, in order.
- *"Who is sending as example.com?"* → `get_domain("example.com")`; list the
  senders by volume with their category; call out `unknown` and `threat`
  senders with their volume and whether their mail was delivered.
- *"Write a report for the board"* → `get_company_report`; it is already
  Markdown written in plain terms; shorten rather than rewrite.
- *"Anything new?"* → `list_alerts`; group by kind, newest first.

## Free tools (no account needed)

The email-authentication tools are open to everyone at
`https://app.dmarclens.io/tools`. They read public DNS only and store nothing,
and they work for any domain, not just ones registered in DMARC Lens.

| Tool | Page | What it does |
|---|---|---|
| Check a domain | `/tools/check?domain=example.com` | Traffic-light verdict on DMARC, SPF and DKIM, with the records and what is wrong |
| DMARC record builder | `/tools/dmarc` | Edits the record a domain publishes, or builds a new one |
| SPF flattener | `/tools/spf` | Resolves includes to IP networks, counts lookups against the limit of ten, splits long records |
| ARC chain checker | `/tools/arc` | Reads a pasted message's ARC headers hop by hop and verifies the seals |
| DKIM key generator | `/tools/dkim` | Makes a key pair in the browser; the private key never leaves it |
| MTA-STS wizard | `/tools/mta-sts` | Builds the policy file and DNS record from the domain's mail servers |
| TLS reporting wizard | `/tools/tls-rpt` | Builds the TLS-RPT record |
| BIMI wizard | `/tools/bimi` | Checks DMARC readiness and builds the record |

**Without a key:** give the user the page link (for the domain check, include
`?domain=` so it runs on arrival). The pages run a short automatic browser
check for visitors and are limited per address; do not script them.

**With an API key** the lookups behind the tools can be called directly, with
the same `Authorization: Bearer dml_…` header. They sit under `/api/tools`
(not `/api/v1`) and return JSON:

```
GET  /api/tools/check?domain=example.com&selector=       full DMARC/SPF/DKIM/MX/MTA-STS/TLS-RPT/BIMI check
GET  /api/tools/dmarc-record?domain=example.com          the DMARC record published today, parsed
GET  /api/tools/spf-flatten?domain=example.com&keep=     flattened SPF; keep = includes to leave in place, comma separated
GET  /api/tools/mail-records?domain=example.com          MX, MTA-STS, TLS-RPT and BIMI records and the MTA-STS policy
GET  /api/tools/dkim-selector?domain=example.com&selector=s1   selectors in use; whether a name is free
GET  /api/tools/report-address?domain=example.com&address=rua@example.net   can this address receive the domain's reports?
POST /api/tools/arc      {"message": "<raw headers or message>"}   the ARC chain, verified
```

Each lookup can take up to 25 seconds, because it waits on other people's DNS.
The tools never change a record: they produce the text to publish, and the
domain's owner publishes it at their DNS provider.

## What this skill does not do

It cannot add domains, approve senders, acknowledge alerts, change plans, or
publish DNS records. Those are done in the app at https://app.dmarclens.io or
at the DNS provider. The DKIM key generator runs only in a browser, by design.
