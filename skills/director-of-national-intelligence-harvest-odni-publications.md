---
name: Harvest the ODNI publications catalog
description: Enumerate ODNI's published documents — transparency reports, Annual Threat Assessments, IC directives, NCTC guides — with their categories and download counts.
api: openapi/director-of-national-intelligence-wp-content-openapi.yml
operations: [listDownloadCategories, listDownloads, getDownload, listMedia]
---

# Harvest the ODNI publications catalog

ODNI's documents are managed by the Download Monitor plugin and exposed as a custom post type on the
public WordPress REST API. This is the closest thing ODNI has to a machine-readable document catalog, and
it is the most useful thing on the API.

## Before your first call

Read the three access rules in `skills/director-of-national-intelligence-track-odni-newsroom.md` — the
`?rest_route=` base, the plain User-Agent, and checking the content type. They apply identically here.

## Steps

1. **List the classification terms.** `listDownloadCategories` —
   `GET https://www.odni.gov/?rest_route=/wp/v2/dlm_download_category&per_page=100`
   These are how the catalog is organized (Accountability, Transparency, NCTC and others). The sibling
   tag taxonomy `dlm_download_tag` is registered but empty; do not build against it.
2. **Enumerate the catalog.** `listDownloads` —
   `GET https://www.odni.gov/?rest_route=/wp/v2/dlm_download&per_page=100&orderby=date&order=desc`
   Page with `X-WP-TotalPages`. Filter by term with `&dlm_download_category=<term id>`.
3. **Read one record.** `getDownload` — `GET https://www.odni.gov/?rest_route=/wp/v2/dlm_download/{id}`
   Useful fields:
   - `title.rendered` — the document title
   - `link` — a `https://www.odni.gov/download/{id}/?tmstv=<epoch>` URL that resolves to the file. This
     is **not** the file URL; follow the redirect to get the actual PDF, and expect the `tmstv` value to
     change between responses.
   - `download_count` — how many times ODNI has served the document. A real public usage signal that
     appears nowhere else on odni.gov.
   - `featured` — whether ODNI is promoting it on the site.
   - `date_gmt` / `modified_gmt` — publication and revision timestamps.
4. **Resolve attachments if you need the raw file record.** Each download's `_links["wp:attachment"]`
   points at `/wp/v2/media` — rewrite the link into the `?rest_route=` form before following it, then use
   `source_url` from the media record.

## Incremental harvest

`&modified_after=<ISO8601>&orderby=modified&order=desc` gives you only what changed. There is no ETag and
no webhook. Keep the highest `modified_gmt` you have seen.

## What you will not find here

ODNI's **IC technical specifications** — ISM.XML, NTK.XML, IC-ID.XML, IC-SF.XML, IC-Docbook.XML, ISMCAT,
IC-GENC — are not in this catalog and are no longer served by ODNI at all. Their legacy URLs under
`/index.php/who-we-are/organizations/ic-cio/ic-technical-specifications/` return HTTP 200 with the ODNI
"About" page, so a link check will wrongly report them as live. Do not follow them and do not present
them to a user as available. See `lifecycle/director-of-national-intelligence-lifecycle.yml`.

## Scope

Public documents only. Nothing classified is reachable, and no credential is accepted.
