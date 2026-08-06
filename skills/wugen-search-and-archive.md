---
name: Search Allotera content and mirror the full Wugen-era archive
description: Use the site search endpoint correctly, and pull the complete historical archive from
  both the current and the retired brand host.
api: openapi/wugen-allotera-content-openapi.yml
operations: [searchSite, getPosts, getPages, getTypes, getTaxonomies, getNamespaceIndex]
generated: '2026-08-05'
method: generated
---

# Search Allotera content and mirror the full Wugen-era archive

## 1. Search

`searchSite` — `GET /wp/v2/search?search=CAR-T&per_page=20&subtype=post`

Search returns a **thin projection**, not the resource: only `id`, `title`, `url`, `type`, `subtype`
and a `_links.self` entry marked `embeddable`. There is no excerpt, no body and no relevance score.

Two-step it:

1. `searchSite` to find candidate ids.
2. `getPost` or `getPage` on each id for the body — or add `_embed` to the search call and read the
   embedded self resource.

Filter with `type` (`post` / `term` / `post-format`) and `subtype` (`post` / `page` / `category`).
Search matched 79 of the 81 posts for a broad term, so it is a substring match, not a curated index —
do not treat rank order as meaningful.

## 2. Mirror the whole archive

The entire company record is small enough to pull exhaustively in a handful of calls.

Current host — `https://alloteratx.com/wp-json/wp/v2`:

- `getPosts` — `?per_page=100&page=1` then `page=2` (81 posts)
- `getPages` — `?per_page=100` (18 pages)
- `getMediaItems` — `?per_page=100` (66 objects)
- `getCategories` — `?per_page=100` (10 terms)

Legacy host — `https://wugen.com/wp-json/wp/v2` (`openapi/wugen-legacy-content-openapi.yml`):

- `getPosts` — `?per_page=100` (80 posts)
- `getMediaItems` — `?per_page=100&page=1..3` (269 objects)
- `getCategories` — `?per_page=100` (11 terms)

Deduplicate across hosts on **title**, not on id — the two installs use independent id sequences and
independent category vocabularies. Expect near-total overlap on posts and heavy divergence on media.

## 3. Confirm the surface before you trust it

`getNamespaceIndex` (`GET /wp/v2`), `getTypes` (`GET /wp/v2/types`) and `getTaxonomies`
(`GET /wp/v2/taxonomies`) are the discovery documents. Use them to check whether a custom post type
has appeared since this profile was written — if the company ever exposes pipeline programs or trial
data as structured content, a new `rest_base` will show up in `getTypes` first.

As of 2026-08-05 the only content-bearing types are `post`, `page` and `attachment`. Everything else
registered (`wp_block`, `wp_template`, `wp_template_part`, `wp_global_styles`, `wp_navigation`,
`wp_font_family`, `elementor_library`, `e-floating-buttons`, `nav_menu_item`) is theme and
page-builder machinery, not company content. Skip it.

## 4. Cautions

- **The retired host has no deprecation signal.** wugen.com serves no `Sunset` header, no
  `Deprecation` header and no API-layer redirect to alloteratx.com. Do not build a durable
  integration against it; it can go dark without notice.
- **wugen.com soft-404s.** Every path outside `/wp-json/` returns HTTP 200 with a 39,987-byte splash
  page. Hash-compare against a control path before believing any 200 from that host.
- `/wp/v2/users`, `/wp/v2/settings`, `/wp/v2/themes` and `/wp/v2/plugins` return HTTP 401. So does
  `/wp-abilities/v1/abilities` on alloteratx.com. These are not reachable; do not retry them with
  invented credentials.
- No rate limits are published and Wordfence fronts both hosts. Serialize bulk pulls and back off on
  429 or an unexpected 403.
