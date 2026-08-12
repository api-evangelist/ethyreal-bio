---
name: Read the Ethyreal Bio leadership and board roster
description: Build the Ethyreal Bio leadership and board-of-directors roster from the team_member custom post type, resolving each person to their Leadership or Board of Directors grouping.
api: openapi/ethyreal-bio-people-api-openapi.yml
operations:
  - getTeamMember
  - getTeamMemberById
  - getTeamGroup
  - getMedia
generated: '2026-08-12'
method: generated
source: Derived from openapi/ethyreal-bio-people-api-openapi.yml and data-model/ethyreal-bio-data-model.yml
---

# Read the Ethyreal Bio leadership and board roster

Ethyreal Bio publishes its executive team and board as a WordPress custom post type
(`team_member`) registered by the Avada theme, grouped by a `team_group` taxonomy. Ten people
were published as of 2026-08-12: four in **Leadership**, seven in **Board of Directors** — eleven
assignments across ten people, so at least one person sits in both groups. Do not assume the
groups partition the roster.

Base URL: `https://www.ethyrealbio.com/wp-json/wp/v2`

## Before you start

- No authentication; read operations are anonymous.
- Send a browser `User-Agent` — Cloudflare bot management returns an HTML 403 otherwise.
- This is personal data about named individuals published by the company itself. Use it as the
  company's own public statement of who works there; do not enrich it against third-party
  people-data brokers.

## Steps

### 1. Resolve the groups first — `getTeamGroup`

```
GET /wp/v2/team_group?per_page=100&_fields=id,name,slug,count
```

Returns the taxonomy terms with member counts, e.g. `{id: 22, name: "Leadership", count: 4}` and
`{id: 23, name: "Board of Directors", count: 7}`. Build an id → name map before step 2 so you can
label people without a second lookup per person.

### 2. List the people — `getTeamMember`

```
GET /wp/v2/team_member?per_page=100&_fields=id,slug,link,title,content,team_group,featured_media
```

- `title.rendered` is the person's name with credentials, e.g. `Nick Williams, MPharm, Ph.D.`
- `content.rendered` is the biography as **HTML**. Strip markup before summarizing.
- `team_group` is an array of term ids — map through step 1.
- `link` is the canonical public profile URL.

Check `X-WP-Total` to confirm you received everyone.

Shortcut: add `_embed` and skip step 1 entirely — the terms arrive inline under
`_embedded['wp:term']`.

### 3. Filter to one group

The taxonomy is queryable directly:

```
GET /wp/v2/team_member?team_group=22&per_page=100
```

### 4. Fetch one profile — `getTeamMemberById`

```
GET /wp/v2/team_member/{id}
```

### 5. Headshots — `getMedia`

`featured_media` is frequently **`0`** on this site: the Avada theme renders portraits through
its own field rather than the WordPress featured image, so the standard link is not reliably
populated. When `featured_media` is non-zero:

```
GET /wp/v2/media/{featured_media}?_fields=id,source_url,alt_text,media_details
```

When it is `0`, do not guess — either parse the `<img>` out of `content.rendered`, or accept that
no portrait is addressable through the API and say so rather than inventing a URL. Headshots do
exist in the media library (57 items, filenames matching people's names), but nothing in the API
binds them to a person, so any such match is inference, not data.

## Error handling

Same `WP_Error` envelope as everywhere on this host — `{code, message, data:{status}}`. See
`errors/ethyreal-bio-problem-types.yml`.

Note that `/wp/v2/users` — the WordPress *author* accounts — is **not** the roster and is closed
anyway: it returns `403 aios_user_lists_forbidden` from the site's security plugin. The roster is
`team_member`. Do not substitute one for the other.

## Do not

- Do not attempt writes. Modifying a company's published leadership page is destructive, requires
  credentials Ethyreal Bio does not issue publicly, and cannot be safely retried — there is no
  idempotency key on this API.
- Do not treat a person's absence from the API as their departure from the company. This is a
  marketing site, updated at the company's convenience.
