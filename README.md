# public.enroll.wisc.edu — Search API Behavior Analysis

Reverse-engineered documentation of the undocumented search API powering the
UW-Madison course enrollment portal at
[https://public.enroll.wisc.edu/search](https://public.enroll.wisc.edu/search).

All endpoints, request bodies, filter-to-Elasticsearch mappings, response
shapes, and client-side behaviors in this folder were reconstructed by
reading the minified frontend bundle (`chunk-PJYL2HPO.js`) and confirming
each path against live DevTools network traffic. There is **no official
public specification** published by UW-Madison for this API.

To the extent that any copyright subsists in this documentation, it is released under CC0 1.0 Universal (Public Domain Dedication). The underlying API structure, field names, and behaviors are factual observations of a publicly accessible system and are not claimed as original authorship.

## Status

- **Base URL:** `https://public.enroll.wisc.edu`
- **Transport:** HTTPS only. JSON request/response bodies.
- **Authentication:** none. Enforces a browser-like `User-Agent`/`Referer`
  (see [06-behaviors-and-quirks.md](./06-behaviors-and-quirks.md)).
- **Rate limiting:** not advertised. No `X-RateLimit-*` headers observed.
- **Content-Type:** `application/json` on all POST bodies.

## Files in this folder

| File | Contents |
|------|----------|
| [01-endpoints.md](./01-endpoints.md) | The 5 confirmed HTTP endpoints, one section each. |
| [02-search-request.md](./02-search-request.md) | `POST /api/search/v1` request body — the Elasticsearch DSL contract. |
| [03-filters-reference.md](./03-filters-reference.md) | Every UI filter, its mapped ES clause, and enum values. |
| [04-response-schemas.md](./04-response-schemas.md) | `CourseHit`, `EnrollmentPackage`, `Details`, `Aggregate`. |
| [05-url-parameters.md](./05-url-parameters.md) | Deep-link query string format on `/search?…`. |
| [06-behaviors-and-quirks.md](./06-behaviors-and-quirks.md) | Auth quirks, merging rules, cache-busting, edge cases. |
| [openapi.yaml](./openapi.yaml) | OpenAPI 3.1 specification (machine-readable). |

## Endpoints at a glance

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/search/v1` | Main course search (Elasticsearch query) |
| `GET`  | `/api/search/v1/aggregate` | Reference data: terms, sessions, specialGroups, subjects |
| `GET`  | `/api/search/v1/subjectsMap/{termCode}` | Subject list for a term (`0000` = all terms) |
| `GET`  | `/api/search/v1/enrollmentPackages/{termCode}/{subjectCode}/{courseId}` | Sections / enrollment packages for one course |
| `GET`  | `/api/search/v1/details/{termCode}/{subjectCode}/{courseId}` | Full course details page payload |

> `subjectCode` in path segments is the **numeric** code (e.g. `"207"`),
> not the abbreviation (`"MATH"`). See the Subject Code Quirk in
> [06-behaviors-and-quirks.md](./06-behaviors-and-quirks.md).

## Reading order

1. Start with [01-endpoints.md](./01-endpoints.md) for the HTTP surface.
2. Read [02-search-request.md](./02-search-request.md) to understand the
   request body format — this is the non-obvious piece.
3. Use [03-filters-reference.md](./03-filters-reference.md) as a lookup
   table while wiring up a client.
4. Consult [06-behaviors-and-quirks.md](./06-behaviors-and-quirks.md)
   before shipping anything that talks to this API.
