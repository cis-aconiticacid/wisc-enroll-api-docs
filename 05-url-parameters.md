# 5. URL Parameters — `/search?…`

The portal renders all current filter state into the browser URL so
searches are shareable. These query parameters are consumed by the
frontend only; the backend API is unaware of them. Documented here
because any well-behaved client should be able to round-trip between a
`SearchFilters` object and a deep link.

## 5.1 Boolean parameters

Encoded as lowercase `"true"` / `"false"`. The client omits any boolean
that equals its default (`false`).

```
open, waitlisted, closed,
biologicalSciences, humanities, literature,
naturalSciences, physicalSciences, socialSciences,
commA, commB, quantA, quantB, ethnicStudies,
elementary, intermediate, advanced,
honorsOnly, acceleratedHonors, honorsOptional,
graduateCourseworkRequirement, workplaceExperience,
communityBasedLearning, repeatableForCredit
```

## 5.2 Enum parameters

| Param              | Values |
|--------------------|--------|
| `modeOfInstruction`| `classroom` \| `hybrid` \| `async` \| `sync` \| `either` (absent = `all`) |
| `language`         | `first` \| `second` \| `third` \| `fourth` \| `fifth` (absent = `all`) |
| `orderBy`          | `subject` \| `catalog-number` (absent = `relevance`) |

## 5.3 Scalar parameters

| Param      | Example         | Notes |
|------------|-----------------|-------|
| `term`     | `1264`          | Term code. |
| `subject`  | `MATH`          | **Short description**, not numeric. UI translates on submit. |
| `keywords` | `calculus`      | URL-encoded free text. |
| `courseId` | `019342`        | Zero-padded 6-char. |
| `topicId`  | `12345`         | Numeric. |

## 5.4 Compound range parameters

The portal packs ranges into one parameter with a `-` separator.

### Credits — `credits=min-max`

Either side may be empty. Examples:

```
credits=3-         min 3, no max
credits=-5         no min, max 5
credits=3-5        between 3 and 5
credits=3-3        exactly 3 (translated to range, not term)
```

### Catalog number — `catalogNum=min-max`

Same pattern. Special cases:

```
catalogNum=220     exact match (min == max, single segment)
catalogNum=220-325 range over zero-padded catalogSort
catalogNum=220-    catalogSort gte "00220"
catalogNum=-325    catalogSort lte "00325"
```

### Sessions — `sessions=CODE1,CODE2`

Comma-separated list of `sessionCode` values. Order is not significant.

```
sessions=YBB,YCC
```

### Reserved sections — `reservedSections=…`

| Form                 | Meaning |
|----------------------|---------|
| (absent)             | all (no filter) |
| `none`               | exclude reserved sections |
| `RESH`               | any value of attribute `RESH` |
| `RESH-BIO`           | attribute `RESH`, value `BIO` |

## 5.5 Round-trip rules

- Empty, default, or `"all"` values are **not** emitted (unless a client
  passes `include_defaults=true`).
- The portal does **not** emit URL params for:
  - `catalogSort` (derived from `catalogNum`)
  - `selectedTerm == "0000"` (treated as "no term")
  - Internal `matched_queries`, `pageSize`, `sortOrder: SCORE`.
- `page` is not in the URL — the portal keeps it in component state.

## 5.6 Example

Filter state:

```python
SearchFilters(
  term="1264",
  subject="MATH",
  advanced=True,
  open=True,
  modeOfInstruction="classroom",
  commA=True,
  creditsMin=3, creditsMax=5,
  catalogNumMin="220", catalogNumMax="325",
  sessions=["YBB", "YCC"],
  reservedSections={"attr": "RESH", "code": "BIO"},
  orderBy="catalog-number",
)
```

Encoded:

```
https://public.enroll.wisc.edu/search
  ?term=1264
  &subject=MATH
  &open=true
  &commA=true
  &advanced=true
  &modeOfInstruction=classroom
  &orderBy=catalog-number
  &credits=3-5
  &sessions=YBB,YCC
  &reservedSections=RESH-BIO
  &catalogNum=220-325
```

(Spaces added for readability; the real URL is a single line.)
