---
name: Track Wugen/Allotera press releases and scientific publications
description: Pull the company's news stream as structured JSON, filtered to press releases or
  scientific publications, and page through it correctly.
api: openapi/wugen-allotera-content-openapi.yml
operations: [getCategories, getPosts, getPost]
generated: '2026-08-05'
method: generated
---

# Track Wugen/Allotera press releases and scientific publications

Allotera Therapeutics (formerly Wugen) publishes its news as WordPress posts. There is no press-release
API product — this is the CMS content API, read anonymously over HTTPS. No credential is required and
none is obtainable.

Base URL: `https://alloteratx.com/wp-json/wp/v2`

## 1. Resolve the category vocabulary first

Post filtering is by integer category id, so you must look the ids up rather than assume them.

`getCategories` — `GET /wp/v2/categories?per_page=100&_fields=id,slug,name,count`

Observed on 2026-08-05: `press-releases` = 15 (59 posts), `scientific-publications` = 14 (12 posts),
`articles` = 16 (8), `careers` = 8 (2), `clinical` = 13 (1). Re-resolve rather than hardcoding —
these ids are install-specific and differ on the legacy host.

## 2. Page the filtered stream

`getPosts` — `GET /wp/v2/posts?categories=15&per_page=100&orderby=date&order=desc&_fields=id,date,modified,slug,link,title,excerpt,categories`

- `per_page` is capped at 100. Asking for more returns HTTP 400 `rest_invalid_param` with the exact
  bound in `data.params.per_page`.
- Read `X-WP-TotalPages` from the first response and stop there, or follow the RFC 8288 `Link` header
  `rel="next"` until it is absent.
- Do **not** page until you get an empty array — requesting a page past the end returns HTTP 400
  `rest_post_invalid_page_number`.
- Use `after=<ISO8601>` or `modified_after=<ISO8601>` for incremental polling instead of refetching
  the archive.

## 3. Fetch a single item when you need the body

`getPost` — `GET /wp/v2/posts/{id}`

`content.rendered` is HTML, not plain text or markdown. `excerpt.rendered` is also HTML. A missing or
unpublished id returns HTTP 404 `rest_post_invalid_id`, which does not distinguish "absent" from
"not publicly viewable".

Add `_embed` to resolve the featured image and terms in the same call. Do not try to resolve
`author` — `/wp/v2/users` returns HTTP 401 `rest_user_cannot_view` on this install, so the author id
is visible but the author is not.

## 4. The legacy archive

The retired `wugen.com` host still serves 80 historical posts at
`https://wugen.com/wp-json/wp/v2/posts` (`openapi/wugen-legacy-content-openapi.yml`). Category ids
are different there. If you need the complete Wugen-era record, pull both hosts and deduplicate on
title, not on id.

**Do not probe wugen.com by path.** That host returns HTTP 200 with a 39,987-byte splash page for
every unmatched path. Only `/wp-json/*` behaves correctly.

## Error handling

All errors are `{code, message, data: {status}}` as `application/json` — not RFC 9457. Branch on
`code`, not on the message string. See `errors/wugen-problem-types.yml`.

No rate limits are published; Wordfence fronts the host. Back off on 429 or an unexpected 403 and do
not run concurrent bulk pulls.
