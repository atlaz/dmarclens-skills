# DMARC Lens for AI agents

How to connect an AI assistant or a script to [DMARC Lens](https://www.dmarclens.io/),
the DMARC report monitoring service: an agent skill, the MCP server and the
read-only API. Everything here is read-only: nothing an agent does through
these can change an account or a DNS record.

## Agent skill

`skills/dmarclens/SKILL.md` follows the open [Agent Skills](https://agentskills.io)
format, read by Claude Code, Codex, Cursor and others. It teaches an agent how
to read a DMARC Lens account, what the fields mean, and what it must not claim.

    npx skills add atlaz/dmarclens-skills@dmarclens

The same file is served at <https://www.dmarclens.io/SKILL.md>.

## MCP server

    https://app.dmarclens.io/api/mcp

Streamable HTTP, read-only, ten tools (`whoami`, `list_companies`,
`get_overview`, `get_company`, `get_company_insights`, `get_company_report`,
`list_domains`, `get_domain`, `get_domain_insights`, `list_alerts`).

- **Claude (claude.ai, desktop):** Settings → Connectors → Add custom
  connector, paste the address, sign in to DMARC Lens and choose Allow.
- **Claude Code:**

      claude mcp add --transport http dmarclens https://app.dmarclens.io/api/mcp

  then run `/mcp` and authenticate.
- **Anything else that speaks MCP:** the server advertises OAuth 2.1 at
  `/.well-known/oauth-protected-resource/api/mcp`, with dynamic client
  registration and PKCE. Clients that cannot do OAuth can send an API key as
  `Authorization: Bearer dml_…` instead.

The connection sees exactly what the person who approved it sees in the app.

## REST API

    https://app.dmarclens.io/api/v1

`GET /api/v1` lists the endpoints and needs no key. Everything else takes a
personal key, created in the app under **API**:

    curl -s -H "Authorization: Bearer $DMARCLENS_KEY" \
      "https://app.dmarclens.io/api/v1/domains/example.com?days=30"

API and MCP access is part of an organisation's plan.

## Free tools

The email-authentication tools need no account:
<https://app.dmarclens.io/tools>: domain check, DMARC record builder, SPF
flattener, ARC chain checker, DKIM key generator, and MTA-STS, TLS reporting
and BIMI wizards. The skill lists the lookups behind them, which an API key can
call directly.

## Licence

MIT. See `LICENSE`.
