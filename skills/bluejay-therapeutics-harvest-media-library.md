---
name: Harvest the Bluejay Therapeutics media library
description: >-
  Enumerate and fetch the 99 files in the Bluejay Therapeutics media library — press-release PDFs,
  conference poster and presentation decks, and site imagery — and tie each file back to the post
  it belongs to.
api: openapi/bluejay-therapeutics-content-openapi.yml
base_url: https://bluejaytx.com/wp-json
authentication: none
operations:
  - listMedia
  - getMediaItem
  - listPosts
  - getPost
  - getOEmbed
generated: '2026-08-07'
method: generated
---

# Harvest the Bluejay Therapeutics media library

## What this is

The 99-item WordPress media library behind bluejaytx.com. It mixes the substantive documents — the
PDFs and conference decks attached to Bluejay Therapeutics' clinical announcements — with ordinary
site chrome: logos, Elementor screenshots, background images.

The company was acquired by Mirum Pharmaceuticals on 26 January 2026 and its marketing pages were
deleted, but the media library and its files are still served. Nothing has been uploaded since
2026-01-26.

## Authentication

None. All operations below return 200 anonymously.

## Step 1 — enumerate the library

`listMedia`:

    GET /wp/v2/media?per_page=100&_fields=id,slug,date,mime_type,media_type,source_url,post,title

`X-WP-Total` returned 99 and `X-WP-TotalPages` returned 10 at the default page size on 2026-08-07.
At `per_page=100` the whole library comes back in one call.

`source_url` is the directly fetchable asset. That is the field you want — not `link`, which points
at the WordPress attachment page.

## Step 2 — isolate the documents that matter

Most of the 99 items are site chrome. Filter by MIME type to get the substantive material:

    GET /wp/v2/media?mime_type=application/pdf&per_page=100&_fields=id,date,title,source_url,post

Or by media type:

    GET /wp/v2/media?media_type=image&per_page=100&_fields=id,source_url,mime_type
    GET /wp/v2/media?media_type=file&per_page=100&_fields=id,source_url,mime_type

Date-window the same way as posts, with `after` / `before` on ISO 8601 timestamps.

## Step 3 — tie a file back to its announcement

Each media item carries a `post` field: the id of the post the file was uploaded to, or `0` for a
library-level upload.

Given a media item with `post: 567`, fetch the announcement with `getPost`:

    GET /wp/v2/posts/567?_fields=id,date,title,link,excerpt

Going the other way, a post's attachments hang off its `_links.wp:attachment` relation, and its
featured image off `_links.wp:featuredmedia`. Follow those links rather than building URLs — the
`curies` entry in `_links` expands the `wp:` prefix to `https://api.w.org/{rel}`.

`getMediaItem` retrieves one file's record:

    GET /wp/v2/media/610

## Step 4 — pick the right image variant

For images, `media_details.sizes` holds the generated variants (thumbnail, medium, large and theme
sizes), each with its own `source_url`, `width` and `height`. `media_details` at the top level
carries the original's width, height and filesize. Choose a variant rather than pulling the
original when you only need a thumbnail.

For non-images, `media_details` is minimal — `filesize` is usually all you get.

## Step 5 — optional, get an embed representation

`getOEmbed` returns the oEmbed 1.0 document for any bluejaytx.com URL — provider name, title,
thumbnail and iframe HTML:

    GET /oembed/1.0/embed?url=https://bluejaytx.com/&format=json&maxwidth=600

The `url` parameter is required. The sibling `/oembed/1.0/proxy` consumer endpoint returns
`401 rest_forbidden` anonymously — do not call it.

## Errors and edge cases

- `404 rest_post_invalid_id` — unknown media id.
- `400 rest_invalid_param` — bad parameter. `per_page` is capped at 100 and exceeding it errors
  rather than clamping. Valid `orderby` values are `author`, `date`, `id`, `include`, `modified`,
  `parent`, `relevance`, `slug`.
- Errors use the WordPress `{code, message, data.status}` envelope over `application/json`, not
  RFC 9457 problem+json. Branch on `code`.
- `POST /wp/v2/media` and `/wp/v2/media/{id}/edit` exist in the route index but are capability
  gated and unreachable. This is a read-only surface.

## Fetching the files themselves

`source_url` values are plain HTTPS URLs under `https://bluejaytx.com/wp-content/uploads/...`,
served by WP Engine over TLS 1.3. They are not behind the REST API and need no special handling.

No rate-limit headers are published; robots.txt asks for a 10-second crawl delay. If you are
pulling all 99 files, throttle.

## Do not

Do not call `/wp/v2/users`, and do not use `?_embed` to resolve the `author` relation on media or
post objects — that collection returns records for named individuals anonymously and is excluded
from this catalogue as personal data. The `author` id on a media item is structural; leave it
unresolved.

## Preservation note

This library belongs to a company that no longer exists independently. There is no deprecation
policy, no Sunset header and no announced end of life for the surface — it can be switched off or
redirected to the acquirer at any time with no notice. If the files matter to you, mirror them.
