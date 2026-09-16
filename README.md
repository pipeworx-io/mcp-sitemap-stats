# @pipeworx/sitemap-stats

Reads a domain's sitemap(s) and reports their STRUCTURE, not their content — URL
counts by path prefix, the fingerprint that separates a normal site from a
programmatic content farm (a farm concentrates a large majority of its URLs
under one or two path prefixes).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

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

Both URLs reach the same gateway and the same 1576+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

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

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-sitemap-stats
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

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
