# @pipeworx/sitemap-stats

Reads a domain's sitemap(s) and reports their STRUCTURE, not their content — URL
counts by path prefix, the fingerprint that separates a normal site from a
programmatic content farm (a farm concentrates a large majority of its URLs
under one or two path prefixes).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1857+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `sitemap_stats({ domain, max_urls? })` — resolves a domain's sitemap(s)
  (robots.txt `Sitemap:` directives, falling back to `/sitemap.xml` then
  `/sitemap_index.xml`), follows one level of sitemap-index nesting, and
  returns `total_urls`, `sitemap_count`, a top-20 path-prefix histogram
  (`top_prefixes`, each with `count` and `share`), `top_prefix_share` (the
  single number that separates a normal site from a generated one), and
  `lastmod_min`/`lastmod_max` where present. Caps total URLs read (default
  150,000, hard ceiling 500,000 via `max_urls`) and always sets `capped: true`
  with a `cap_reason` when it stops early, so a truncated read never looks
  like a small site. A domain with no sitemap returns `found: false` with a
  `reason` — not an error.

## Auth

Keyless. No vendor, no key.

## Data sources

- The target domain's own `robots.txt` and sitemap XML files, read live on
  every call — a direct pass-through, not a copy.

## Traps found while building this

- Sitemap protocol caps a single file at 50,000 URLs, so real multi-file sites
  are common — the tool follows a `<sitemapindex>` one level down to its child
  `<sitemap>` files, but does not recurse further (nested indexes-of-indexes
  are non-standard and rare).
- A raw `.xml.gz` served as a static file carries no `Content-Encoding: gzip`
  header, so the runtime's automatic decompression never triggers — decoded by
  hand via Workers-native `DecompressionStream('gzip')` when the URL ends in
  `.gz` or the response's `content-type` mentions gzip.
- The URL cap applies to total URLs read across every file, not per file, and
  a run that hits it stops mid-file rather than skipping whole files — so
  `total_urls` under a cap is always the true count of what was actually read,
  never silently smaller than that.
- `robots.txt` `Sitemap:` lines are supposed to be absolute per spec but
  aren't always in the wild; relative ones are resolved against the domain's
  origin.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "sitemap-stats": {
      "url": "https://gateway.pipeworx.io/sitemap-stats/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/sitemap-stats/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1857+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/sitemap_stats \
  -H 'Content-Type: application/json' \
  -d '{"domain":"gitnux.org"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/sitemap_stats`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-sitemap-stats` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/sitemap-stats/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "sitemap-stats": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-sitemap-stats"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Sitemap Stats data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
