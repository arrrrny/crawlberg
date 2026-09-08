The `crawlberg` CLI is a high-performance command-line tool for web scraping, crawling, URL discovery, and browser interaction. It wraps the core Rust engine and exposes every major operation as a subcommand — from scraping a single page to crawling thousands of URLs concurrently. This page covers installation, every command and its options, advanced configuration via JSON, output formats, and diagnostic controls.

**What you'll learn**: How to invoke each `crawlberg` subcommand, configure it with flags and JSON, and interpret results on stdout. After reading, you'll be ready to move to the hands-on guides in [Scrape a Single Page](6-scrape-a-single-page), [Crawl a Website](7-crawl-a-website), and [Discover URLs with Map](8-discover-urls-with-map).

## CLI Overview

The binary name is `crawlberg`. Every invocation follows a consistent pattern:

```text
crawlberg [--global-options] <COMMAND> [command-args] [command-options]
```

Subcommands map directly to the core library's public API — `scrape`, `crawl`, `map_urls`, `batch_scrape`, `batch_crawl`, and `interact` — plus convenience commands for the REST API server, MCP server, document downloading, citation generation, and version reporting.

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L209-L226)

## Global Options

These flags appear **before** the subcommand and apply to every operation:

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--log-level <LEVEL>` | — | — | Explicit log level: `off`, `error`, `warn`, `info`, `debug`, or `trace`. Overrides `-v`/`-q` when set. |
| `--log-format <FORMAT>` | — | `pretty` | Output format for diagnostic logs on stderr: `pretty`, `compact`, or `json`. |
| `--verbose` | `-v` | — | Increase verbosity. `-v` = info, `-vv` = debug, `-vvv` = trace. Default is warn-level. |
| `--quiet` | `-q` | — | Suppress all diagnostics below error level. |

**Log-level precedence**: `--log-level` wins over `-v`/`-q`. When neither is given, the default is `warn` with an additional `crawlberg_cli=info` rule so the CLI's own progress messages are visible. Diagnostics always go to **stderr**; command results go to **stdout**.

You can also override log filtering at runtime via environment variables `CRAWLBERG_LOG` (preferred) or `RUST_LOG`, which take precedence over the built-in directives.

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L12-L82), [logging.rs](crates/crawlberg-cli/src/telemetry/logging.rs#L67-L86)

## Commands

### scrape — Single Page Extraction

Scrape a single URL and extract metadata, links, images, structured data (JSON-LD, Open Graph, Dublin Core, Twitter Card), and markdown content.

```text
crawlberg scrape <URL> [OPTIONS]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `URL` | Yes | The URL to scrape |

| Option | Default | Description |
|--------|---------|-------------|
| `--format <FORMAT>` | `json` | Output format: `json` or `markdown` |
| `--proxy <URL>` | — | Proxy URL (HTTP, HTTPS, or SOCKS5) |
| `--user-agent <STRING>` | — | Custom User-Agent header |
| `--timeout <MS>` | `30000` | Request timeout in milliseconds |
| `--respect-robots-txt` | `false` | Honor robots.txt directives |
| `--browser-mode <MODE>` | `auto` | When to use the headless browser: `auto`, `always`, or `never` |
| `--browser-endpoint <WS_URL>` | — | CDP WebSocket URL for an external browser (must start with `ws://` or `wss://`) |
| `--config <JSON>` | — | Configuration as a JSON string or `@file.json` reference |

**Examples:**

```bash
# Basic JSON scrape
crawlberg scrape https://example.com

# Markdown output
crawlberg scrape https://example.com --format markdown

# Through a proxy with extended timeout
crawlberg scrape https://example.com --proxy http://proxy:8080 --timeout 60000

# Force browser rendering for a JavaScript-heavy SPA
crawlberg scrape https://quotes.toscrape.com/js/ --browser-mode always --format markdown

# Advanced config via JSON string
crawlberg scrape https://example.com --config '{"download_assets":true,"max_asset_size":50000}'
```

**Output behavior**: In `json` mode the full `ScrapeResult` is printed as pretty-printed JSON. In `markdown` mode only the markdown content is printed; if no markdown was generated (e.g., a PDF link), a warning goes to stderr.

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L231-L258), [main.rs](crates/crawlberg-cli/src/main.rs#L501-L551)

---

### crawl — Follow Links Across a Site

Crawl a website by following internal links up to a configurable depth. Accepts one seed URL (single-crawl mode) or multiple seed URLs (batch-crawl mode — when two or more URLs are given, `batch_crawl` is invoked automatically).

```text
crawlberg crawl <URL>... [OPTIONS]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `URL` | Yes (one or more) | Seed URL(s) to crawl. Multiple URLs trigger batch mode. |

| Option | Short | Default | Description |
|--------|-------|---------|-------------|
| `--depth <N>` | `-d` | `2` | Maximum crawl depth (0 = only the seed page) |
| `--max-pages <N>` | `-n` | — | Total page budget; crawl stops when reached |
| `--concurrent <N>` | `-c` | `10` | Maximum concurrent HTTP requests |
| `--rate-limit <MS>` | — | `200` | Minimum delay between requests to the same domain (ms) |
| `--format <FORMAT>` | — | `json` | Output format: `json` or `markdown` |
| `--proxy <URL>` | — | — | Proxy URL |
| `--user-agent <STRING>` | — | — | Custom User-Agent header |
| `--timeout <MS>` | — | `30000` | Request timeout in milliseconds |
| `--respect-robots-txt` | — | `false` | Honor robots.txt directives |
| `--stay-on-domain` | — | `false` | Restrict crawling to the seed domain |
| `--browser-mode <MODE>` | — | `auto` | Browser fallback: `auto`, `always`, or `never` |
| `--browser-endpoint <WS_URL>` | — | — | CDP WebSocket URL for external browser |
| `--config <JSON>` | — | — | JSON configuration string or `@file.json` |

**Examples:**

```bash
# Default crawl (depth 2, 10 concurrent)
crawlberg crawl https://example.com

# Deep crawl with budget
crawlberg crawl https://example.com -d 5 -c 20 --max-pages 500

# Markdown output, stay on domain
crawlberg crawl https://example.com --format markdown --stay-on-domain

# Batch crawl multiple seeds
crawlberg crawl https://example.com https://example.org -d 1

# Advanced crawl with JSON config
crawlberg crawl https://example.com -d 3 --config '{"exclude_paths":["/admin/.*"],"include_paths":["/blog/.*"]}'
```

**Single vs. batch output**: When one URL is given, `json` mode prints a `CrawlResult` and `markdown` mode prints each page separated by `---` headers. When multiple URLs are given, both modes emit batch aggregate results with `seed_url`, `results`, `total_count`, `completed_count`, and `failed_count`.

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L259-L303), [main.rs](crates/crawlberg-cli/src/main.rs#L552-L623)

---

### map — Discover URLs

Discover all URLs on a website via sitemaps (XML, sitemap index, robots.txt `Sitemap:` directive) and link extraction, without downloading full page content.

```text
crawlberg map <URL> [OPTIONS]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `URL` | Yes | The site URL to map |

| Option | Default | Description |
|--------|---------|-------------|
| `--limit <N>` | — | Maximum number of URLs to return |
| `--search <STRING>` | — | Filter discovered URLs by substring match |
| `--respect-robots-txt` | `false` | Honor robots.txt directives |
| `--format <FORMAT>` | `json` | Output format: `json` or `markdown` |
| `--timeout <MS>` | `30000` | Request timeout in milliseconds |
| `--browser-mode <MODE>` | `auto` | Browser fallback: `auto`, `always`, or `never` |
| `--browser-endpoint <WS_URL>` | — | CDP WebSocket URL for external browser |
| `--config <JSON>` | — | JSON configuration string or `@file.json` |

**Examples:**

```bash
# Discover all URLs on a site
crawlberg map https://example.com

# Limit results and filter
crawlberg map https://example.com --limit 50 --search "/docs/"

# JSON config to exclude paths
crawlberg map https://example.com --config '{"exclude_paths":["/private/.*"]}'
```

**Output**: In `json` mode, a `MapResult` object is printed with a `urls` array. In `markdown` mode, one URL per line is printed to stdout.

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L304-L332), [main.rs](crates/crawlberg-cli/src/main.rs#L624-L668)

---

### interact — Browser Action Execution

Execute a sequence of browser actions (click, type, press key, scroll, take screenshot, execute JavaScript, wait for selector) on a single page. Requires the headless browser.

```text
crawlberg interact <URL> --actions <JSON> [OPTIONS]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `URL` | Yes | The page URL to interact with |

| Option | Default | Description |
|--------|---------|-------------|
| `--actions <JSON>` | — | Actions as a JSON array of `PageAction` objects |
| `--format <FORMAT>` | `json` | Output format: `json` or `markdown` |
| `--timeout <MS>` | `30000` | Request timeout in milliseconds |
| `--browser-mode <MODE>` | `auto` | Browser fallback (typically `always`) |
| `--browser-endpoint <WS_URL>` | — | CDP WebSocket URL for external browser |
| `--config <JSON>` | — | JSON configuration string or `@file.json` |

**Example — fill a form and submit:**

```bash
crawlberg interact https://example.com/form \
  --actions '[
    {"type":"type","selector":"#email","text":"user@example.com"},
    {"type":"type","selector":"#password","text":"secret"},
    {"type":"click","selector":"button[type=submit]"}
  ]' \
  --browser-mode always --format json
```

In `markdown` mode the final HTML is printed. In `json` mode the result is wrapped as `{ "interaction": { ... } }`.

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L333-L355), [main.rs](crates/crawlberg-cli/src/main.rs#L669-L715)

---

### batch-scrape — Concurrent Multi-URL Scraping

Scrape multiple URLs concurrently with configurable concurrency.

```text
crawlberg batch-scrape <URL>... [OPTIONS]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `URL` | Yes (one or more) | URLs to scrape concurrently |

| Option | Short | Default | Description |
|--------|-------|---------|-------------|
| `--concurrent <N>` | `-c` | `10` | Maximum concurrent requests |
| `--format <FORMAT>` | — | `json` | Output format: `json` or `markdown` |
| `--proxy <URL>` | — | — | Proxy URL |
| `--user-agent <STRING>` | — | — | Custom User-Agent header |
| `--timeout <MS>` | — | `30000` | Request timeout in milliseconds |
| `--respect-robots-txt` | — | `false` | Honor robots.txt directives |
| `--browser-mode <MODE>` | — | `auto` | Browser fallback: `auto`, `always`, or `never` |
| `--browser-endpoint <WS_URL>` | — | — | CDP WebSocket URL for external browser |
| `--config <JSON>` | — | — | JSON configuration string or `@file.json` |

**Examples:**

```bash
# Scrape 3 pages concurrently
crawlberg batch-scrape https://a.com https://b.com https://c.com -c 5

# Markdown output with JSON config
crawlberg batch-scrape https://a.com https://b.com --format markdown --config '{"download_assets":true}'
```

In `markdown` mode each result is separated by `---` with a `URL:` header. In `json` mode the aggregate `BatchScrapeResults` object is printed with per-URL results and counts.

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L356-L388), [main.rs](crates/crawlberg-cli/src/main.rs#L716-L758)

---

### batch-crawl — Concurrent Multi-Seed Crawling

Crawl multiple seed URLs concurrently, each following links independently.

```text
crawlberg batch-crawl <URL>... [OPTIONS]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `URL` | Yes (one or more) | Seed URLs to crawl concurrently |

| Option | Short | Default | Description |
|--------|-------|---------|-------------|
| `--depth <N>` | `-d` | `2` | Maximum crawl depth per seed |
| `--max-pages <N>` | `-n` | — | Maximum pages to crawl per seed |
| `--concurrent <N>` | `-c` | `10` | Maximum concurrent requests |
| `--rate-limit <MS>` | — | `200` | Rate limit delay per domain (ms) |
| `--format <FORMAT>` | — | `json` | Output format: `json` or `markdown` |
| `--proxy <URL>` | — | — | Proxy URL |
| `--user-agent <STRING>` | — | — | Custom User-Agent header |
| `--timeout <MS>` | — | `30000` | Request timeout in milliseconds |
| `--respect-robots-txt` | — | `false` | Honor robots.txt directives |
| `--stay-on-domain` | — | `false` | Stay on the same domain as each seed |
| `--browser-mode <MODE>` | — | `auto` | Browser fallback: `auto`, `always`, or `never` |
| `--browser-endpoint <WS_URL>` | — | — | CDP WebSocket URL for external browser |
| `--config <JSON>` | — | — | JSON configuration string or `@file.json` |

**Example:**

```bash
crawlberg batch-crawl https://blog.example.com https://docs.example.com -d 2 -c 15 --format json
```

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L389-L433), [main.rs](crates/crawlberg-cli/src/main.rs#L759-L809)

---

### download — Fetch Document Metadata

Download a document (PDF, DOCX, etc.) from a URL and report its metadata (MIME type, size, filename, content hash) without printing the binary content.

```text
crawlberg download <URL> [OPTIONS]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `URL` | Yes | URL to download |

| Option | Default | Description |
|--------|---------|-------------|
| `--max-size <BYTES>` | — | Maximum document size in bytes |
| `--timeout <MS>` | `30000` | Request timeout in milliseconds |
| `--browser-mode <MODE>` | `auto` | Browser fallback: `auto`, `always`, or `never` |
| `--browser-endpoint <WS_URL>` | — | CDP WebSocket URL for external browser |
| `--config <JSON>` | — | JSON configuration string or `@file.json` |

**Example:**

```bash
crawlberg download https://example.com/report.pdf --max-size 10485760
```

If the URL returns a downloadable document, the output includes `url`, `mime_type`, `size`, `filename`, and `content_hash`. If the URL returns HTML instead, a note is printed.

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L434-L453), [main.rs](crates/crawlberg-cli/src/main.rs#L810-L861)

---

### citations — Link-to-Citation Conversion

Convert markdown-style links into numbered citation references. Accepts inline text or a file path (prefixed with `@`).

```text
crawlberg citations <INPUT>
```

| Argument | Required | Description |
|----------|----------|-------------|
| `INPUT` | Yes | Markdown text or `@file.md` to read from a file |

**Example:**

```bash
# Inline
crawlberg citations 'See [this article](https://example.com) for details.'

# From file
crawlberg citations @notes.md
```

Output is a JSON object containing the processed markdown and citation data.

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L454-L458), [main.rs](crates/crawlberg-cli/src/main.rs#L862-L876)

---

### version — Print Version

Print the crawlberg version as JSON. Useful for health checks and scripting.

```text
crawlberg version
```

**Output:** `{"version":"1.1.2"}`

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L459-L460), [main.rs](crates/crawlberg-cli/src/main.rs#L877-L880)

---

### serve — REST API Server

Start the HTTP REST API server. Requires the `api` feature at build time.

```text
crawlberg serve [OPTIONS]
```

| Option | Default | Description |
|--------|---------|-------------|
| `--host <ADDRESS>` | `0.0.0.0` | Host address to bind to |
| `--port <PORT>` | `3000` | Port to listen on |

**Example:**

```bash
crawlberg serve --host 127.0.0.1 --port 8080
```

When built with both `api` and `mcp` features, the MCP Streamable HTTP transport is also available at `/mcp`. For details, see [REST API Server](25-rest-api-server).

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L462-L470), [main.rs](crates/crawlberg-cli/src/main.rs#L881-L890)

---

### mcp — MCP Server for AI Agents

Start the Model Context Protocol server for AI agent integration. Requires the `mcp` feature.

```text
crawlberg mcp [--http] [OPTIONS]
```

| Option | Default | Description |
|--------|---------|-------------|
| `--http` | `false` | Use Streamable HTTP transport instead of stdio (requires `mcp-http` feature) |
| `--host <ADDRESS>` | `127.0.0.1` | Host address in `--http` mode |
| `--port <PORT>` | `3001` | Port in `--http` mode |

**Examples:**

```bash
# stdio mode (default — for Claude Desktop integration)
crawlberg mcp

# HTTP mode
crawlberg mcp --http --host 127.0.0.1 --port 3001
```

For Claude Desktop integration, add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "crawlberg": {
      "command": "crawlberg",
      "args": ["mcp"]
    }
  }
}
```

For details, see [MCP Server for AI Agents](24-mcp-server-for-ai-agents).

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L471-L484), [main.rs](crates/crawlberg-cli/src/main.rs#L891-L915)

## JSON Configuration via `--config`

Most subcommands accept a `--config <JSON>` flag that merges advanced settings into the command's default `CrawlConfig`. This provides access to every configuration field not directly exposed as a CLI flag.

The value can be either:

1. **An inline JSON string**: `--config '{"max_depth":3,"stay_on_domain":true}'`
2. **A file reference** (prefixed with `@`): `--config @my-config.json`

JSON values **override defaults** but do **not** override CLI flags that were explicitly set on the command line. This means you can put all your advanced settings in a config file and override individual values with flags as needed.

**Common `--config` examples:**

```bash
# Crawl with include/exclude path patterns
crawlberg crawl https://example.com --config '{"exclude_paths":["/admin/.*"],"include_paths":["/blog/.*"]}'

# Enable asset downloading with size limits
crawlberg scrape https://example.com --config '{"download_assets":true,"max_asset_size":50000}'

# Allow subdomains and custom headers
crawlberg map https://example.com --config '{"allow_subdomains":true,"custom_headers":{"Accept-Language":"en-US"}}'

# Enable WARC recording
crawlberg crawl https://example.com --config '{"warc_output":"/tmp/crawl.warc"}'

# Use a stored config file
crawlberg crawl https://example.com --config @production-config.json
```

The full list of available JSON fields is documented in the [Configuration Reference](11-configuration-reference-crawlconfig).

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L132-L160)

## Output Formats

Every command that produces results supports two output modes:

| Format | Behavior |
|--------|----------|
| `json` (default) | Prints the full structured result as pretty-printed JSON to **stdout** |
| `markdown` | Prints human-readable markdown content to **stdout** |

In `markdown` mode, batch operations separate results with `---` YAML front matter headers:

```
---
Seed: https://example.com
URL: https://example.com/page-1
---

# Page 1 content in markdown...

---
URL: https://example.com/page-2
---

# Page 2 content in markdown...
```

All diagnostic messages (errors, warnings, progress) go to **stderr** via the `tracing` subscriber, keeping stdout clean for machine-parseable data.

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L84-L89), [main.rs](crates/crawlberg-cli/src/main.rs#L162-L207)

## Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Error — scrape/crawl/map failure, invalid configuration, or server startup failure |

Error details are printed to stderr. The process exits with code 1 on any failure in the core pipeline, invalid JSON config, or server errors.

Sources: [main.rs](crates/crawlberg-cli/src/main.rs#L546-L550)

## Diagnostics and Telemetry

### Console Logging

The CLI installs a global `tracing` subscriber that writes diagnostics to stderr. You can control verbosity three ways:

```bash
# Verbose mode: -v (info), -vv (debug), -vvv (trace)
crawlberg scrape https://example.com -vv

# Quiet mode: only errors
crawlberg scrape https://example.com -q

# Explicit level via flag
crawlberg scrape https://example.com --log-level debug

# Or via environment variable
CRAWLBERG_LOG=debug crawlberg scrape https://example.com
```

### OTLP Export (Feature-Gated)

When built with the `otel` feature and the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable is set, spans and metrics are exported to an OTLP collector in addition to console logging. If OTLP initialization fails, the CLI falls back to console logging automatically.

Sources: [mod.rs](crates/crawlberg-cli/src/telemetry/mod.rs#L31-L54), [logging.rs](crates/crawlberg-cli/src/telemetry/logging.rs#L67-L86)

## Feature Flags

The CLI binary is compiled with optional feature flags that control which subcommands are available:

| Feature | Enables |
|---------|---------|
| `api` | `serve` subcommand and REST API server |
| `mcp` | `mcp` subcommand with stdio transport |
| `mcp-http` | `mcp --http` with Streamable HTTP transport |
| `warc` | WARC recording support via JSON config |
| `ai` | AI-powered content extraction |
| `otel` | OpenTelemetry OTLP export |
| `all` | All of the above |

The default pre-built binary includes all features. If you build from source, use `--features all` for the full experience or pick individual features.

Sources: [Cargo.toml](crates/crawlberg-cli/Cargo.toml#L21-L39)

## Next Steps

- **Hands-on tutorials**: [Scrape a Single Page](6-scrape-a-single-page), [Crawl a Website](7-crawl-a-website), [Discover URLs with Map](8-discover-urls-with-map)
- **Advanced configuration**: [Configuration Reference (CrawlConfig)](11-configuration-reference-crawlconfig)
- **Architecture deep-dive**: [Architecture and Data Flow](9-architecture-and-data-flow)
- **AI agent integration**: [MCP Server for AI Agents](24-mcp-server-for-ai-agents)
- **REST API usage**: [REST API Server](25-rest-api-server)
- **Full CLI flag reference**: [CLI Reference](4-cli-usage) (auto-generated)