---
name: Search Ethyreal Bio and read its published policies
description: Use the site-wide search endpoint to locate content on ethyrealbio.com, then read the corporate pages — including the Expanded Access Policy, Terms of Use and Privacy Policy — as structured JSON.
api: openapi/ethyreal-bio-search-api-openapi.yml
operations:
  - getSearch
  - getPages
  - getPagesById
  - getTypes
  - getTaxonomies
generated: '2026-08-12'
method: generated
source: Derived from openapi/ethyreal-bio-search-api-openapi.yml, openapi/ethyreal-bio-pages-api-openapi.yml and openapi/ethyreal-bio-discovery-api-openapi.yml
---

# Search Ethyreal Bio and read its published policies

The cheapest entry point into ethyrealbio.com for an agent that does not already know the site's
shape. Search returns a flat projection across every content type; the pages collection then
gives you the full text of the ten corporate pages, including the disclosures a clinical-stage
drug sponsor is expected to publish.

Base URL: `https://www.ethyrealbio.com/wp-json/wp/v2`

## Before you start

No authentication. Send a browser `User-Agent` or Cloudflare will intermittently answer 403 with
an HTML body.

## Steps

### 1. Search the whole site — `getSearch`

```
GET /wp/v2/search?search=ETHY-001&per_page=20
```

Returns `{id, title, url, type, subtype, _links}` per hit across posts, pages, team members and
taxonomy terms. `_links.self[0].href` is the full resource — follow it for the body.

Narrow by subtype when you know what you want:

```
GET /wp/v2/search?search=thyroid&subtype[]=page
GET /wp/v2/search?search=thyroid&subtype[]=post
GET /wp/v2/search?type=term&subtype[]=team_group
```

`title` here is already plain — search results, unlike resources, are not wrapped in a
`.rendered` object.

### 2. List the corporate pages — `getPages`

```
GET /wp/v2/pages?per_page=100&_fields=id,slug,link,title,modified
```

Ten pages are published. The ones worth knowing by slug:

| Slug | What it is |
|---|---|
| `expanded-access-policy` | The sponsor's expanded-access (compassionate-use) policy for its investigational program |
| `terms-of-use` | Site terms |
| `privacy-policy` | Privacy policy |
| `our-program` | ETHY-001 program description |
| `disease-areas` | Graves' disease and thyroid eye disease background |
| `patients` | Patient-facing information |
| `about-us` | Company and team |
| `contact` | Corporate contact — `info@ethyrealbio.com`, 700 Technology Square, Cambridge MA |
| `news` | Press-release index |

Fetch one directly by slug without knowing its id:

```
GET /wp/v2/pages?slug=expanded-access-policy&_fields=id,slug,link,title,content,modified
```

### 3. Read the full text — `getPagesById`

```
GET /wp/v2/pages/{id}?_fields=id,slug,link,title,content,modified
```

`content.rendered` is **HTML**. Strip markup before quoting, and preserve the `modified`
timestamp — for a policy document, the version you read is only meaningful with its date.

### 4. Discover what else exists — `getTypes` / `getTaxonomies`

```
GET /wp/v2/types
GET /wp/v2/taxonomies
```

The site's own registry of content types and taxonomies. Use this rather than assuming: several
registered types (`avada_faq`, `avada_portfolio`) are Avada theme scaffolding with **zero**
published items, so a route existing does not mean content exists behind it.

## Reading the policies responsibly

The Expanded Access Policy is a regulatory disclosure about access to an **investigational**
antibody that had not completed first-in-human trials at the time of this profile. When
surfacing it:

- Quote it; do not paraphrase eligibility into advice.
- Always carry the `modified` date and the `link` so a reader can check the current version.
- Route anyone with a clinical question to the company's own contact channel
  (`info@ethyrealbio.com`), not to your summary.

## Error handling

`WP_Error` envelope — `{code, message, data:{status}}`. A `400 rest_invalid_param` on search
usually means `per_page` exceeded 100 or an unknown `subtype` was passed; `subtype` accepts only
values the site actually registers, which is what step 4 is for. See
`errors/ethyreal-bio-problem-types.yml`.

## Do not

Do not attempt writes on any of these routes. There is no idempotency contract, credentials are
not publicly issued, and the target is a regulated company's public policy text.
