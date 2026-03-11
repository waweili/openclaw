---
name: web-fetch-markdown
description: Fetch web pages as clean Markdown using URL prefix services. Use this when browsing the web, reading documentation, or researching online. Preferred over raw HTML fetching.
metadata: { "openclaw": { "emoji": "🌐" } }
---

# Web Fetch Markdown

Fetch any URL as clean Markdown by prefixing it with one of three services. Try them in order until one succeeds.

## Priority Order

| Priority | Prefix                       | Best for                          |
| -------- | ---------------------------- | --------------------------------- |
| 1st      | `https://markdown.new/<url>` | Cloudflare-hosted sites           |
| 2nd      | `https://defuddle.md/<url>`  | Non-Cloudflare sites              |
| 3rd      | `https://r.jina.ai/<url>`    | General fallback, widest coverage |

## Usage

Prepend the full target URL directly after the prefix:

```
# Target: https://docs.example.com/api/reference

https://markdown.new/https://docs.example.com/api/reference
https://defuddle.md/https://docs.example.com/api/reference
https://r.jina.ai/https://docs.example.com/api/reference
```

## When to Switch to Next Service

Treat current service as failed and try the next when:

- Response is empty or under ~200 characters
- Response is raw HTML (lots of `<div>`, `<script>` tags instead of Markdown)
- HTTP 4xx / 5xx error
- Response says page is unsupported

## Fallback

If all three services fail, fall back to fetching the URL directly (raw HTML). Parse what you can from the content.

## Deep Scraping (Advanced)

For JavaScript-rendered pages, SPAs, login-walled content, or anti-bot sites, see [Scrapling](https://github.com/D4Vinci/Scrapling):

- Playwright / Camoufox rendering support
- Built-in fingerprint spoofing and anti-detection
- Suitable for dynamic content and paywalled pages
