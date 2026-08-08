---
name: Read the Bluejay Therapeutics press and publication archive
description: >-
  Retrieve, filter and read the 35 press releases and publication notices Bluejay Therapeutics
  published between 2021 and 2025, over the anonymous WordPress REST API that outlived the
  company's web site.
api: openapi/bluejay-therapeutics-content-openapi.yml
base_url: https://bluejaytx.com/wp-json
authentication: none
operations:
  - listPosts
  - getPost
  - listCategories
  - getCategory
  - listTags
  - search
generated: '2026-08-07'
method: generated
---

# Read the Bluejay Therapeutics press and publication archive

## What this is

Bluejay Therapeutics was a clinical-stage biopharmaceutical company developing brelovitug (BJT-778)
for chronic hepatitis delta and B. Mirum Pharmaceuticals completed its acquisition on 26 January
2026 and the corporate site was reduced to a single acquisition notice — `/about/`, `/science/`,
`/pipeline/` and `/news/` all return 404.

The REST API was left intact. All 35 press releases and publication notices are still readable,
with full text. This skill is how you get at them.

The dataset is **frozen**: the newest post is dated 2025-12-08. Do not treat it as a live feed.

## Authentication

None. Every operation below returns 200 without any credential. Send no `Authorization` header.

## Step 1 — get the archive index cheaply

Full post objects run to roughly 15 KB each because `content.rendered` carries the entire press
release. Always pull the index with a sparse fieldset first.

`listPosts`:

    GET /wp/v2/posts?per_page=100&_fields=id,slug,date,title,link,categories,tags

That single call returns all 35 items. Confirm with the `X-WP-Total` response header (35) and
`X-WP-TotalPages` (1 at `per_page=100`).

`_fields` is honoured by the WordPress controller but is **not** declared in the site's route
index, so it will be missing from any spec generated purely from that index. It works.

## Step 2 — know your categories before filtering

`listCategories`:

    GET /wp/v2/categories?per_page=100&_fields=id,name,slug,count

The four terms and their post counts, verified 2026-08-07:

| id | name | slug | count |
|----|------|------|-------|
| 14 | Press Release | `press-releases` | 26 |
| 17 | Publication | `publication` | 8 |
| 15 | Presentation | `presentations` | 0 |
| 1 | Uncategorized | `uncategorized` | 1 |

`count` is the cheapest way to size a slice without paging it. Use `getCategory` for a single term.

There is one tag only — id 16, "Corporate Presentation", 1 post — via `listTags`.

## Step 3 — filter to the slice you want

Press releases only:

    GET /wp/v2/posts?categories=14&per_page=100&_fields=id,slug,date,title,link

Scientific publications only:

    GET /wp/v2/posts?categories=17&per_page=100&_fields=id,slug,date,title,link

A date window (ISO 8601):

    GET /wp/v2/posts?after=2025-01-01T00:00:00&per_page=100&_fields=id,date,title,link

Ordering is `orderby=date&order=desc` by default. Other valid `orderby` values are `author`, `id`,
`include`, `modified`, `parent`, `relevance` and `slug`; anything else returns 400
`rest_invalid_param`.

## Step 4 — search across the archive

`search` is a federated projection returning `{id, title, url, type, subtype}` rather than full
objects:

    GET /wp/v2/search?search=brelovitug&per_page=100

    GET /wp/v2/search?search=hepatitis&per_page=100

A `search=hepatitis` query returned 31 matches on 2026-08-07. Note `title` here is a plain string,
unlike the `{rendered: ...}` object on the underlying post. Resolve a hit by fetching the
collection named in `subtype` at the given `id`.

For matching inside post bodies rather than the search projection, use the `search` parameter on
`listPosts` instead, optionally with `search_semantics=exact` to disable fuzzy behaviour.

## Step 5 — read a single item

`getPost`:

    GET /wp/v2/posts/579

Post 579 is the December 2025 acquisition announcement — a useful smoke test.

Before you parse the body:

- `content.rendered` is **HTML**, not markdown. Newer posts are Elementor-authored and arrive
  wrapped in `div[data-elementor-type="wp-post"]` scaffolding around the prose; older posts render
  as clean `wp-block-*` markup. Strip both.
- `title.rendered` and `excerpt.rendered` carry HTML entity escaping (`&#8212;`, `&#039;`).
- Use `link` for the permalink. **Never resolve `guid.rendered`** — it is a frozen internal
  identifier of the form `https://bluejaytxstg.wpenginepowered.com/?p={id}` pointing at a WP Engine
  staging host. It does not resolve.
- `excerpt_length` on `getPost` overrides the word count of the rendered excerpt.

## Errors and edge cases

Errors use the WordPress envelope `{code, message, data.status}` over `application/json`. This is
**not** RFC 9457 — there is no `type` URI, so branch on `code`.

- `404 rest_post_invalid_id` — unknown or unpublished id. List the collection and use ids from it.
- `400 rest_invalid_param` — a parameter failed its schema. Offending params are enumerated under
  `data.params`. `per_page` is capped at 100 and exceeding it errors rather than clamping.
- `401 rest_forbidden` — you have wandered off the public surface. Nothing under
  `/wp/v2/settings`, `/wp/v2/themes`, `/wp/v2/plugins`, `aioseo/v1`, `elementor/v1` or
  `wp-abilities/v1` is reachable anonymously, and no credential is obtainable by a third party.
- An empty array is a **success**. Do not retry it.

## Rate limiting

No `RateLimit` or `Retry-After` headers are published. The only signal the provider gives is
`Crawl-delay: 10` in robots.txt. Self-throttle; one request per second is more than polite for a
35-item archive that changes never.

## Do not

Do not call `/wp/v2/users`. It returns author records for named individuals anonymously. It is
excluded from this catalogue as personal data and no tool is provided for it. Post objects carry an
integer `author` id and an `_links.author` relation; leave them unresolved.

Do not use `?_embed` on this API for the same reason — it inlines author records into the payload.
If you need media, follow `_links.wp:featuredmedia` or use the media skill instead.
