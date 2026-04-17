# 2. Search Request Body — `POST /api/search/v1`

This is the non-obvious part of the API. The portal builds an
Elasticsearch **bool query** on the client and ships it as the `filters`
array. The server appears to forward it almost verbatim to an ES index.

## 2.1 Top-level shape

```jsonc
{
  "selectedTerm": "1264",   // string. "0000" = search across all terms.
  "queryString":  "*",      // full-text query; "*" = match all
  "filters":      [ … ],    // array of ES clauses, see §2.3
  "page":         1,        // 1-indexed
  "pageSize":     50,       // server caps at 50; requesting higher silently clamps
  "sortOrder":    "SCORE"   // SCORE | SUBJECT | CATALOG_NUMBER
}
```

| Field          | Required | Type   | Notes |
|----------------|----------|--------|-------|
| `selectedTerm` | yes      | string | `"0000"` enables the "all terms" mode — disables seat and session filters. |
| `queryString`  | yes      | string | Free text matched against title / description / subject / catalog number. Empty string → use `"*"`. |
| `filters`      | yes      | array  | See §2.3. May be empty. |
| `page`         | yes      | int    | Starts at 1. Out-of-range pages return `hits: []` with the original `found`. |
| `pageSize`     | yes      | int    | Portal always sends `50`. |
| `sortOrder`    | yes      | enum   | See §2.4. |

## 2.2 Term semantics — `selectedTerm`

- `"0000"` is a sentinel meaning **all terms**. The portal uses it on the
  landing page before any term is picked.
- When `"0000"` is set, the server silently ignores filters that rely on
  term-specific state: seats, sessions, the published flag, reserved
  sections, etc. See `has_term` gating in
  [03-filters-reference.md](./03-filters-reference.md).
- When a real term is set (e.g. `"1264"`), the portal always adds a
  `has_child(published:true)` clause to suppress hidden sections. Do the
  same.

## 2.3 The `filters` array

Each element is an Elasticsearch clause. Observed clause types:

- `{"term":  {"field": "value"}}` — exact match on keyword fields.
- `{"terms": {"field": ["v1", "v2"]}}` — any-of exact match.
- `{"match": {"field": "value"}}` — analyzed text match.
- `{"range": {"field": {"gte": x, "lte": y}}}` — range on numeric /
  zero-padded string fields.
- `{"bool":  {"should": [...], "must": [...], "must_not": [...]}}` —
  composition.
- `{"query": {"exists": {"field": "name"}}}` — presence check (wrapped in
  `query` for historic reasons).
- `{"has_child": {"type": "enrollmentPackage", "query": {...}}}` —
  joins into the child index of section data. Most enrollment-state and
  section-level filters live here.

> **Critical merging rule:** the portal collapses every top-level
> `has_child(enrollmentPackage)` filter into **one** `has_child` whose
> inner `query` is a `bool.must` of the individual clauses. Sending
> multiple sibling `has_child` filters will parse but returns empty /
> wrong results, because child-join scoring is per-clause. See
> [06-behaviors-and-quirks.md §6.3](./06-behaviors-and-quirks.md#63-hash_child-merging).

### 2.3.1 Example — "Open, Advanced, COM-A, Classroom" for Spring 2026

```json
{
  "selectedTerm": "1264",
  "queryString": "*",
  "filters": [
    { "bool": { "should": [
        { "terms": { "generalEd.code": [["COM A"]] } }
    ]}},
    { "terms": { "levels.code": ["A"] } },
    { "has_child": {
        "type": "enrollmentPackage",
        "query": { "bool": { "must": [
          { "match": { "packageEnrollmentStatus.status": "OPEN" } },
          { "bool": {
              "must": [{ "match": { "modesOfInstruction": "Instruction" } }],
              "must_not": [
                { "match": { "modesOfInstruction": "some" } },
                { "match": { "modesOfInstruction": "Only" } },
                { "match": { "modesOfInstruction": "EMPTY" } }
              ]
          }},
          { "match": { "published": true } }
        ]}}
    }}
  ],
  "page": 1, "pageSize": 50, "sortOrder": "SCORE"
}
```

Note the three child-level filters (seats, mode of instruction,
published) merged into one `has_child`.

## 2.4 `sortOrder` enum

| Value            | UI label         | ES behavior |
|------------------|------------------|-------------|
| `SCORE`          | Relevance        | `_score` descending; ties broken by `subject.subjectCode`, then `catalogSort`. |
| `SUBJECT`        | Subject          | `subject.subjectCode` ascending, then `catalogSort`. |
| `CATALOG_NUMBER` | Catalog number   | `catalogSort` ascending (zero-padded, 5 chars). |

`catalogSort` is a string key — sort is lexical over zero-padded catalog
numbers (`"00220"` < `"00325"` < `"01010"`).

## 2.5 Pagination

- Requests are 1-indexed. `page=1` returns hits `[0..pageSize)`.
- `found` is the total result count across all pages; it does **not**
  change as you page.
- A page past the end returns `hits: []`, `success: true`. There is no
  error; clients must check `len(hits) == 0 || len(hits_so_far) >= found`.
- There is no cursor / `search_after` support.

## 2.6 Response shape

```json
{
  "found": 5238,
  "hits": [ /* CourseHit */ ],
  "message": null,
  "success": true
}
```

`success` has been `true` in every 2xx response observed. `message` is
used to surface ES parse errors in 400 / 500 responses.
