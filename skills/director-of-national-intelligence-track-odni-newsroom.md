---
name: Track the ODNI newsroom
description: Pull ODNI press releases, remarks, interviews and reports as structured records, and poll for new ones without re-reading the site.
api: openapi/director-of-national-intelligence-wp-content-openapi.yml
operations: [listCategories, listPosts, getPost]
---

# Track the ODNI newsroom

ODNI has no developer program. Its newsroom is available as structured JSON anyway, through the
WordPress REST API behind www.odni.gov. No credential is needed.

## Before your first call

1. **Do not use `https://www.odni.gov/wp-json/`.** That base and everything under it is answered with
   HTTP 301 to the site homepage, which returns 200 and 187KB of HTML. Use the query-string route form:
   `https://www.odni.gov/?rest_route=/wp/v2/posts`
2. **Do not send a browser User-Agent.** The Akamai edge answers 403 to browser-shaped UA strings and
   serves plain clients normally.
3. **Check the content type** on every response. `text/html` means you were redirected to the homepage,
   not that the record is missing.

## Steps

1. **Learn the categories.** `listCategories` — `GET https://www.odni.gov/?rest_route=/wp/v2/categories&per_page=100`
   Four are in use: `press-releases` (id 4), `reports` (id 5), `remarks` (id 6), `interviews` (id 118).
   Term ids are not sequential; do not guess them, and note that id 1 returns 404.
2. **List the posts you want.** `listPosts` —
   `GET https://www.odni.gov/?rest_route=/wp/v2/posts&categories=4&per_page=100&orderby=date&order=desc`
   Read `X-WP-Total` and `X-WP-TotalPages` from the response headers, then walk `page=2..N`.
   `per_page` is capped at 100 — 101 or more returns HTTP 400 `rest_invalid_param`.
3. **Trim the payload.** Add `&_fields=id,date_gmt,modified_gmt,slug,link,title,categories` to avoid
   pulling the full rendered HTML of every article when you only need a list.
4. **Fetch one article in full.** `getPost` — `GET https://www.odni.gov/?rest_route=/wp/v2/posts/{id}`
   `content.rendered` is HTML, not plain text.

## Polling for new items

There is no ETag, no Last-Modified and no webhook, and `Cache-Control` is `private, no-cache`. The only
reliable change signal is the record's own timestamp:

```
GET https://www.odni.gov/?rest_route=/wp/v2/posts&modified_after=2026-09-01T00:00:00&orderby=modified&order=desc
```

Store the highest `modified_gmt` you have seen and pass it as `modified_after` next time. The RSS feed at
`https://www.odni.gov/feed/` is a lighter alternative when you only need new publications.

Be polite about frequency. ODNI publishes no rate limits and returns no rate-limit headers, so you get no
warning before the edge decides you are abusive. The site's robots.txt asks non-SearchGov crawlers for a
10-second delay; treat that as the intended pace.

## Errors

Errors are the WordPress envelope, not RFC 9457: `{"code": "...", "message": "...", "data": {"status": N}}`.
`rest_invalid_param` (400) names the bad parameter under `data.params`. `rest_post_invalid_id` (404) means
the id does not exist or is not public. Full catalog with reproducing requests:
`errors/director-of-national-intelligence-problem-types.yml`.

## Scope

This is public-affairs content. No intelligence holdings, reporting or classified material is reachable
through this API, and nothing here is authenticated — the write and administrative routes the platform
advertises return 401 to an anonymous caller and are not part of this skill.
