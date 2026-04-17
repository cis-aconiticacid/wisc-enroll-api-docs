# 6. Behaviors and Quirks

Non-obvious runtime behaviors of the search API. Read before shipping
any client — most of these will silently produce wrong results rather
than fail loudly.

## 6.1 403 Forbidden without browser headers

The server rejects requests that look like a script. Requests with:

- curl's default `User-Agent: curl/8.x`
- Python `requests` default `python-requests/2.x`
- no `Referer` or `Origin`

…get an HTML `403 Forbidden` body, not JSON. Sending a realistic
`User-Agent` plus:

```
Referer: https://public.enroll.wisc.edu/search
Origin:  https://public.enroll.wisc.edu
```

…is sufficient. No cookies, no CSRF token, no auth header needed.
Observed headers on the wire:

```
Content-Type: application/json
Accept: application/json, text/plain, */*
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36
Referer: https://public.enroll.wisc.edu/search
Origin:  https://public.enroll.wisc.edu
```

## 6.2 Subject code quirk

The UI displays **abbreviations** (`MATH`, `COMP SCI`) but the API
filters on **numeric** codes (`"600"`, `"266"`).

- Source of truth: `aggregate.subjects[termCode]` (or `subjectsMap/{termCode}`).
- In a `CourseHit`, `subject.subjectCode` is numeric and
  `subject.shortDescription` is the abbreviation.
- Path segments on `/details/{term}/{subject}/{course}` and
  `/enrollmentPackages/{term}/{subject}/{course}` are numeric.
- URL param `subject=MATH` on `/search?…` is the **abbreviation** — the
  frontend translates before calling the API.

Translation table is term-specific: `MATH` may have different numeric
codes across archived terms (rare but observed on historical data).
Always translate against the user-selected term's subject map.

## 6.3 `has_child` merging

Multiple top-level `{"has_child": {"type": "enrollmentPackage", …}}`
clauses in `filters` are **not equivalent** to a single combined one.
The portal deliberately folds all such clauses into **one** `has_child`
whose inner query is a `bool.must` of the originals:

```jsonc
// WRONG — works syntactically, returns incorrect results
"filters": [
  { "has_child": { "type": "enrollmentPackage", "query": A }},
  { "has_child": { "type": "enrollmentPackage", "query": B }},
  { "has_child": { "type": "enrollmentPackage", "query": C }}
]

// RIGHT — always combine
"filters": [
  { "has_child": { "type": "enrollmentPackage",
      "query": { "bool": { "must": [ A, B, C ] }}}}
]
```

Reason: each `has_child` scores against its own best child, so
`seat_status=OPEN` might match section 1 while `mode=classroom` matches
section 2 — the parent course passes both filters even though no single
section satisfies both. Merging constrains the child join to a single
section per course.

## 6.4 Term sentinel `"0000"`

When `selectedTerm == "0000"`:

- Seat filters (Open/Waitlisted/Closed) are **ignored** server-side.
- The implicit `published: true` clause is **not** emitted.
- Session filters pass through but typically match nothing, because
  session codes are term-scoped.
- Reserved section filters behave the same way — generally no-ops.

Treat `"0000"` as "catalog-browsing mode" rather than "search everything
including seats across all terms."

## 6.5 Page size clamp

`pageSize > 50` is silently clamped to 50. No error is returned. Plan
pagination for ≤50 hits per page.

## 6.6 `page` past the end

Paging past `ceil(found / pageSize)` returns:

```json
{ "found": 5238, "hits": [], "success": true, "message": null }
```

No error. Stop iteration on `len(hits) == 0`.

## 6.7 `catalogSort` lexical ordering

Catalog number range uses `catalogSort`, a **zero-padded 5-character
string**. Range values must also be zero-padded:

```json
{ "range": { "catalogSort": { "gte": "00220", "lte": "00325" } } }
```

Without padding, `"220"` sorts after `"1010"` lexically and the range
selects the wrong rows.

## 6.8 `queryString` defaults

An empty `queryString` is invalid; the portal rewrites blank to `"*"`.
`"*"` is a match-all and returns `found` equal to the term's course
count after other filters.

## 6.9 Response caching

- `/aggregate` payload changes only when the registrar publishes new
  term / subject / session data (a few times per semester). Safe to
  cache for hours / days. No `Cache-Control` or `ETag` is sent.
- `/details` and `/enrollmentPackages` change with seat counts; cache
  for seconds to a minute at most if you care about accuracy.
- `/api/search/v1` should not be cached beyond a request — the portal
  itself never reuses prior results.

## 6.10 Inconsistent detail response

`GET /details/…` sometimes returns a single object and sometimes a
1-element array. This appears to reflect ES `_source` projection on a
multi-result query. Normalize:

```python
data = resp.json()
if isinstance(data, list):
    data = data[0] if data else {}
```

## 6.11 Filter clauses that allow-list (not exclude)

`Open/Waitlisted/Closed` act as an **allow list**. If none are checked,
**all** sections (regardless of status) pass. Checking only `OPEN`
excludes waitlisted and closed. The UI hides this by forcing at least
one checkbox when the Seats group is expanded; a programmatic client
needs to decide explicitly.

## 6.12 Published flag and historical terms

`published: true` is enforced for current / future terms. For archived
terms the flag may be absent on some records; the portal still sends
the clause but the server silently ignores it. Don't rely on
`published` being present in hits returned from old terms.

## 6.13 Stability

- **Filter names (`commA`, `honorsOnly`, …)** come from the JS bundle.
  They have not changed across the last few deploys but are not
  officially stable.
- **ES field names (`breadths.code`, `sections.sessionCode`, …)** are
  part of the undocumented backend contract and can change with index
  rebuilds. Treat them as brittle.
- **The path `/api/search/v1`** carries a version segment; a future
  `v2` is possible. None exists today.

## 6.14 Rate limiting

No explicit rate limit was observed. A single client pulling `/aggregate`
once and running 10 searches per second for several minutes did not get
throttled. This is not a license to hammer — be conservative.

## 6.15 Robots.txt

`https://public.enroll.wisc.edu/robots.txt` was checked and does not
block `/api/search/v1`. Nonetheless, identify your client honestly in
`User-Agent` for production use — the DevTools-style fake UA above is
only needed to bypass the 403 gate.
