---
name: Read Allotera corporate, pipeline and patient-facing pages
description: Retrieve the company's pipeline, science, clinical-trials, expanded-access and
  leadership pages as JSON, and understand what the API does not expose.
api: openapi/wugen-allotera-content-openapi.yml
operations: [getPages, getPage, getMediaItems]
generated: '2026-08-05'
method: generated
---

# Read Allotera corporate, pipeline and patient-facing pages

Base URL: `https://alloteratx.com/wp-json/wp/v2`

## 1. List the pages

`getPages` — `GET /wp/v2/pages?per_page=100&_fields=id,slug,title,link,parent,menu_order`

18 pages exist as of 2026-08-05: `about-us`, `science`, `pipeline`, `scientific-publications`,
`press-releases`, `clinical-trials`, `patients`, `expanded-access-policy`, `career-opportunities`,
`contact-us`, `journey`, `leadership`, `board-of-directors`, `partners`, `investors`,
`privacy-notice`, `accessibility`, and the home page.

## 2. Fetch a page by slug

The slug is the stable handle; ids are not stable across the two hosts.

`getPages` — `GET /wp/v2/pages?slug=pipeline&_fields=id,slug,title,content,modified`

This returns a one-element array, not an object. If you already hold an id, `getPage`
(`GET /wp/v2/pages/{id}`) returns the object directly.

## 3. Know the ceiling of this surface

**The therapeutic pipeline is not structured data.** `content.rendered` is Elementor-generated HTML.
There is no program, indication, trial-phase or trial-arm entity anywhere on this API. The `acf`
(Advanced Custom Fields) block is present on every resource and is empty on every object sampled —
posts, pages, media and categories alike.

So: to get "which programs are in which phase", you must parse the HTML of `/pipeline/`. Treat that
as scraping with all its fragility, and re-verify against the rendered page rather than caching a
parse. Do not present a parsed pipeline as if the company published it as data.

For clinical-trial facts, prefer the registry of record (ClinicalTrials.gov) over this HTML. For
expanded-access requests, `/expanded-access-policy/` is the page the company points patients and
physicians to.

## 4. Media

`getMediaItems` — `GET /wp/v2/media?per_page=100&_fields=id,slug,title,media_type,mime_type,source_url,post`

66 objects on alloteratx.com; 269 on the legacy wugen.com host, which remains the fuller image
archive. `source_url` is the direct asset URL. Use `post` to tie an asset back to the page or post it
was uploaded to.

## Conventions and errors

Pagination, `_fields`, `_embed` and the error envelope are identical to the press-release skill — see
`conventions/wugen-conventions.yml` and `errors/wugen-problem-types.yml`. Every operation here is a
GET; there is no reachable write path.
