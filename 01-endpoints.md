# 1. Endpoints

Base URL: `https://public.enroll.wisc.edu`

All endpoints live under `/api/search/v1`. Five endpoints were observed;
no others appear to be exercised by the portal UI.

---

## 1.1 `POST /api/search/v1`

Main course search. Accepts an Elasticsearch-shaped query body; returns
a paginated list of `CourseHit` objects.

**Request**

```http
POST /api/search/v1 HTTP/1.1
Host: public.enroll.wisc.edu
Content-Type: application/json
Accept: application/json, text/plain, */*
Referer: https://public.enroll.wisc.edu/search
Origin:  https://public.enroll.wisc.edu

{ "selectedTerm": "1264", "queryString": "*", "filters": [],
  "page": 1, "pageSize": 50, "sortOrder": "SCORE" }
```

See [02-search-request.md](./02-search-request.md) for the full body
contract and [03-filters-reference.md](./03-filters-reference.md) for
filter clauses.

**Response 200** — `application/json`

```json
{
  "found": 5238,
  "hits":  [ /* CourseHit, up to pageSize */ ],
  "message": null,
  "success": true
}
```

**Errors**

| Status | Cause |
|--------|-------|
| `403`  | Missing browser-like `User-Agent` or `Referer`. |
| `400`  | Malformed body (non-JSON, missing `selectedTerm`, unknown `sortOrder`). |
| `500`  | Usually a malformed ES clause — e.g. a `has_child` on a term without enrollment data. |

---

## 1.2 `GET /api/search/v1/aggregate`

Returns reference data the UI needs for its filter menus: the list of
terms, per-term sessions, per-term subjects, and `specialGroups`.

**Request**

```http
GET /api/search/v1/aggregate HTTP/1.1
```

No query parameters. The payload is large (~1 MB) and rarely changes;
the portal caches it client-side for the life of the page.

**Response 200** — shape

```jsonc
{
  "terms": [
    { "termCode": "1264", "longDescription": "Spring 2025-2026", … }
  ],
  "sessions": [
    { "termCode": "1264", "sessions": [ /* Session */ ] }
  ],
  "subjects": {
    "1264": [ { "subjectCode": "600", "shortDescription": "MATH", … } ],
    "0000": [ … /* union across all terms */ … ]
  },
  "specialGroups": [ /* list-of-list, ordered by term */ ]
}
```

See [04-response-schemas.md §4](./04-response-schemas.md#4-aggregate) for
field-level detail.

---

## 1.3 `GET /api/search/v1/subjectsMap/{termCode}`

Returns the subjects available in a given term, keyed by numeric
`subjectCode`.

| Path segment | Type   | Example | Notes |
|--------------|--------|---------|-------|
| `termCode`   | string | `1264`  | Use `0000` to get the union across all terms. |

**Response 200** — shape

```jsonc
{
  "600": {
    "subjectCode": "600",
    "shortDescription": "MATH",
    "formalDescription": "MATHEMATICS",
    "undergraduateCatalogURI": "…",
    "graduateCatalogURI": "…",
    "departmentURI": "…",
    "schoolCollege": { … },
    "footnotes": [],
    "departmentOwnerAcademicOrgCode": "…"
  },
  "266": { … }
}
```

> The data here is a subset of what `aggregate.subjects[termCode]` returns;
> prefer `aggregate` when you need everything and `subjectsMap` when you
> only need one term's subjects.

---

## 1.4 `GET /api/search/v1/enrollmentPackages/{termCode}/{subjectCode}/{courseId}`

Returns the list of **enrollment packages** (section groupings) for a
single course. Called when the user expands a course card.

| Path segment  | Type   | Example      | Notes |
|---------------|--------|--------------|-------|
| `termCode`    | string | `1264`       | Must be a real term. `0000` is invalid here. |
| `subjectCode` | string | `600`        | **Numeric** code, not `"MATH"`. |
| `courseId`    | string | `019342`     | From `hit.courseId`. Zero-padded. |

**Response 200** — array of `EnrollmentPackage`. See
[04-response-schemas.md §2](./04-response-schemas.md#2-enrollmentpackage).

**Errors**

| Status | Cause |
|--------|-------|
| `404`  | Course not offered in that term, or wrong `subjectCode` format. |

---

## 1.5 `GET /api/search/v1/details/{termCode}/{subjectCode}/{courseId}`

Returns the full details page payload for one course — richer than the
`CourseHit` returned from search (adds footnotes, full descriptions,
cross-listed subject objects, catalog URIs, etc.).

Same path-segment contract as `enrollmentPackages`.

**Response 200** — `Details` object, see
[04-response-schemas.md §3](./04-response-schemas.md#3-details).

---

## Calling-order patterns observed

The portal makes these calls in this order during a typical user flow:

1. Page load → `GET /aggregate`
2. User picks a term → `GET /subjectsMap/{term}` (if not already in
   aggregate cache)
3. User types a query / toggles a filter → `POST /api/search/v1`
4. User clicks a result card → `GET /details/{term}/{subj}/{courseId}`
   **and** `GET /enrollmentPackages/{term}/{subj}/{courseId}` in
   parallel.
5. User pages down → another `POST /api/search/v1` with `page: 2, 3, …`

There are no PUT / PATCH / DELETE endpoints. The API is read-only from
the public surface.
