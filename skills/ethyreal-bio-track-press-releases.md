---
name: Track Ethyreal Bio press releases
description: Read Ethyreal Bio's press releases from the ethyrealbio.com content API, resolve their categories, and detect new announcements without re-fetching everything.
api: openapi/ethyreal-bio-news-api-openapi.yml
operations:
  - getPosts
  - getPostsById
  - getCategories
generated: '2026-08-12'
method: generated
source: Derived from openapi/ethyreal-bio-news-api-openapi.yml and conventions/ethyreal-bio-conventions.yml
---

# Track Ethyreal Bio press releases

Ethyreal Bio is a clinical-stage biotech with no developer program. Its press releases are
readable as JSON only because the marketing site runs WordPress. There is no product API here,
no API key, no support channel, and no stability commitment — treat this as a content feed you
are tolerated on, not one you are entitled to.

Base URL: `https://www.ethyrealbio.com/wp-json/wp/v2`

## Before you start

- **No authentication.** Read operations are anonymous. Do not send an Authorization header.
- **Send a browser `User-Agent`.** The host sits behind Cloudflare bot management. A non-browser
  UA gets an intermittent **HTTP 403 with an HTML body**, not a JSON error. This is the most
  common failure and it is not a permissions problem.
- **The corpus is tiny.** Two press releases as of 2026-08-12. Do not paginate defensively at
  scale; do not poll aggressively. `robots.txt` asks for a 10-second crawl delay.

## Steps

### 1. List the press releases — `getPosts`

```
GET /wp/v2/posts?per_page=100&orderby=date&order=desc&_fields=id,date,modified,slug,link,title,excerpt,categories
```

Read `X-WP-Total` and `X-WP-TotalPages` from the response headers to size the collection before
deciding to paginate. Follow the `Link: <…>; rel="next"` header rather than incrementing `page`
by hand.

`title.rendered` and `excerpt.rendered` are **entity-escaped HTML**, not plain text. Unescape and
strip markup before summarizing, or you will emit `&#8230;` into your output.

### 2. Fetch the full body only for the releases you care about — `getPostsById`

```
GET /wp/v2/posts/{id}?_fields=id,date,modified,link,title,content
```

`content.rendered` is the full release as HTML.

### 3. Resolve the category ids — `getCategories`

```
GET /wp/v2/categories?per_page=100&_fields=id,name,slug,count
```

Posts carry `categories` as an array of integer ids. Category `20` is `News` and carries the
press releases. Alternatively skip this step entirely by adding `_embed` to step 1, which inlines
the terms under `_embedded['wp:term']`.

### 4. Detect new releases on a later run

Two options, cheapest first:

- **Conditional request.** The API returns `Last-Modified` and `Cache-Control: max-age=600`.
  Send `If-Modified-Since` and accept a `304`.
- **Date bound.** `GET /wp/v2/posts?after=<ISO 8601 of your last run>&orderby=date&order=desc`
  returns only what is newer. Persist the highest `modified` timestamp you have seen, not the
  highest `id`.

There is no webhook, no event stream and no AsyncAPI — polling is the only mechanism available.

## Error handling

Errors are the WordPress `WP_Error` envelope — `{code, message, data:{status}}` — **not** RFC 9457
problem+json. Branch on `code`, and check `Content-Type` before parsing:

| Status | `code` | What to do |
|---|---|---|
| 400 | `rest_invalid_param` | Fix the named parameter. `per_page` caps at 100. |
| 404 | `rest_post_invalid_id` | The id does not exist. Re-list to get valid ids. |
| 404 | `rest_no_route` | Wrong path *or wrong method* — check the `Allow` header. |
| 403 | HTML body, `server: cloudflare` | Bot challenge. Back off, retry with a browser UA. Not a quota error. |

Full catalogue: `errors/ethyreal-bio-problem-types.yml`.

## Do not

- **Do not attempt writes.** `POST`/`PUT`/`PATCH`/`DELETE` exist on these routes but require a
  WordPress Application Password that Ethyreal Bio does not issue publicly. There is also **no
  idempotency key** on this API, so a retried write would create duplicate content on a public
  company website. `targetHints.allow` in each resource's `_links` confirms `["GET"]` for you.
- **Do not treat this surface as stable.** Its shape follows the WordPress and Avada theme
  configuration. Re-read `https://www.ethyrealbio.com/wp-json/` rather than trusting a cached
  spec, and expect routes to appear or vanish with no notice.
