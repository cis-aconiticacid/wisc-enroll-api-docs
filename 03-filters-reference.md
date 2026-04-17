# 3. Filters Reference

Every filter exposed in the UI and its exact mapping to an Elasticsearch
clause in the `filters` array of `POST /api/search/v1`.

All mappings derive from the minified JS bundle (`chunk-PJYL2HPO.js`);
the single-letter names in parentheses (`Sr`, `br`, …) are the original
function identifiers, kept here because they anchor the mapping to the
source of truth.

## 3.0 Filter-to-clause summary

| # | UI filter | JS fn | Shape | Gated by `term`? |
|---|-----------|-------|-------|------------------|
| 1 | Seats (Open/Waitlisted/Closed) | `Sr` | `has_child → packageEnrollmentStatus.status` | **yes** |
| 2 | Subject                        | `br` | `term subject.subjectCode` | no |
| 3 | General Education              | `Tr` | `bool.should` of multiple sub-clauses | no |
| 4 | Breadth                        | `Ir` | `terms breadths.code` | no |
| 5 | Level                          | `Or` | `terms levels.code` | no |
| 6 | Foreign Language               | `Er` | `query.match foreignLanguage.code` | no |
| 7 | Honors                         | `$r` | `has_child → sections.honors` | no |
| 8 | Reserved Sections              | `wr` | `has_child → sections.classAttributes` | **yes** |
| 9 | Mode of Instruction            | `Cr` | `has_child → modesOfInstruction` bool composition | no |
| 10 | Course Attributes             | `Fr` `Ar` `xr` `_r` | various | no |
| 11 | Credits                       | `kr` `qr` | `range minimumCredits` / `range maximumCredits` | no |
| 12 | Sessions                      | `zr` | `has_child → sections.sessionCode` | no |
| 13 | Course ID                     | `Pr` | `term courseId` | no |
| 14 | Catalog Number range          | `Lr` | `range catalogSort` (zero-padded) | no |
| 15 | Topic ID                      | `Rr` | `term topics.id` | no |
| 16 | Published (implicit)          | `jr` | `has_child → published:true` | **yes** |

Filters marked "gated by `term`" are silently dropped when
`selectedTerm == "0000"`.

---

## 3.1 Seats — `packageEnrollmentStatus.status`

The UI exposes three checkboxes. Any combination builds a
space-separated list.

| Checkbox    | Value       |
|-------------|-------------|
| Open        | `OPEN`      |
| Waitlisted  | `WAITLISTED`|
| Closed      | `CLOSED`    |

```json
{ "has_child": {
    "type": "enrollmentPackage",
    "query": { "match": { "packageEnrollmentStatus.status": "OPEN WAITLISTED" }}
}}
```

> The JS actually joins the statuses with a space into a single string.
> It's analyzed by ES and behaves like OR. Do not send `terms`.

## 3.2 Subject — `subject.subjectCode`

```json
{ "term": { "subject.subjectCode": "600" } }
```

`subjectCode` is the **numeric** code. The UI shows the abbreviation
(`MATH`, `COMP SCI`) and translates on submit using
`/aggregate.subjects[term]`. See the Subject Code Quirk in
[06-behaviors-and-quirks.md §6.2](./06-behaviors-and-quirks.md#62-subject-code-quirk).

## 3.3 General Education — composite

The Gen-Ed block wraps multiple sub-clauses in `bool.should`:

| UI check    | Sub-clause |
|-------------|------------|
| `commA`     | `{"terms": {"generalEd.code": [["COM A"]]}}` (nested array is intentional) |
| `quantA`    | same shape with `"QR-A"` |
| `quantB`    | same shape with `"QR-B"` |
| `commB`     | `{"has_child": {"type": "enrollmentPackage", "query": {"match": {"sections.comB": true}}}}` |
| `ethnicStudies` | `{"term": {"ethnicStudies.code": "ETHNIC ST"}}` |

COM-B is special: it's a **section-level** property, so it pushes into
`has_child`. Ethnic Studies is **not** part of Gen-Ed in the data model
even though the UI groups it nearby — it lives on `ethnicStudies.code`.

## 3.4 Breadth — `breadths.code`

| UI checkbox        | Code |
|--------------------|------|
| biologicalSciences | `B`  |
| humanities         | `H`  |
| literature         | `L`  |
| naturalSciences    | `N`  |
| physicalSciences   | `P`  |
| socialSciences     | `S`  |

```json
{ "terms": { "breadths.code": ["B", "N"] } }
```

## 3.5 Level — `levels.code`

| UI checkbox   | Code |
|---------------|------|
| elementary    | `E`  |
| intermediate  | `I`  |
| advanced      | `A`  |

```json
{ "terms": { "levels.code": ["A"] } }
```

## 3.6 Foreign Language — `foreignLanguage.code`

| UI value   | Code |
|------------|------|
| `all`      | — (no clause emitted) |
| `first`    | `FL1` |
| `second`   | `FL2` |
| `third`    | `FL3` |
| `fourth`   | `FL4` |
| `fifth`    | `FL5` |

```json
{ "query": { "match": { "foreignLanguage.code": "FL2" } } }
```

## 3.7 Honors — `sections.honors`

| UI checkbox       | Code                 |
|-------------------|----------------------|
| honorsOnly        | `HONORS_ONLY`        |
| acceleratedHonors | `HONORS_LEVEL`       |
| honorsOptional    | `INSTRUCTOR_APPROVED`|

- One box checked → `has_child` with a single `match`.
- Multiple boxes → `has_child` with `bool.should` of matches.

## 3.8 Reserved Sections — `sections.classAttributes`

Three modes, triggered by the UI control value `reservedSections`:

- `"all"` → no clause (default).
- `"none"` → emit `has_child` with a `bool.should` that keeps only
  sections with no `classAttributes` **or** the `TEXT SVCL` attribute.
- `{attr, code}` → include only sections with a specific reserved
  attribute. `code` null = any value of that attribute.

```json
// reservedSections = {"attr": "RESH", "code": "BIO"}
{ "has_child": {
    "type": "enrollmentPackage",
    "query": { "match": { "sections.classAttributes.valueCode": "BIO" }}
}}
```

## 3.9 Mode of Instruction — `modesOfInstruction` + `isAsynchronous`

Six UI values. Each produces a `has_child` with a `bool` composition:

| UI value   | `must`                                            | `must_not`                                               |
|------------|---------------------------------------------------|----------------------------------------------------------|
| `all`      | —                                                 | —                                                        |
| `classroom`| `modesOfInstruction: Instruction`                 | `some`, `Only`, `EMPTY`                                  |
| `hybrid`   | `modesOfInstruction: some`                        | `Instruction`, `Only`, `EMPTY`                           |
| `async`    | `modesOfInstruction: Only` + `isAsynchronous: true` | `Instruction`, `some`, `EMPTY`, `isAsynchronous: false` |
| `sync`     | `modesOfInstruction: Only` + `isAsynchronous: false`| `Instruction`, `some`, `EMPTY`, `isAsynchronous: true`  |
| `either`   | `modesOfInstruction: Only`                        | `Instruction`, `some`, `EMPTY`                           |

Behavior note: these compositions rely on `modesOfInstruction` being a
multi-valued keyword — `"some"` and `"Only"` are sentinel tokens the
index emits for hybrid and fully-online sections.

## 3.10 Course Attributes — flat top-level

| UI checkbox                      | ES clause |
|----------------------------------|-----------|
| graduateCourseworkRequirement    | `{"term": {"gradCourseWork": true}}` |
| workplaceExperience              | `{"query": {"exists": {"field": "workplaceExperience"}}}` |
| communityBasedLearning           | `has_child → sections.classAttributes.valueCode: "25 PLUS"` |
| repeatableForCredit              | `{"match": {"repeatable": "Y"}}` |

## 3.11 Credits — `minimumCredits` / `maximumCredits`

```json
{ "range": { "minimumCredits": { "gte": 3 } } }
{ "range": { "maximumCredits": { "lte": 5 } } }
```

`creditsMin` sets a floor on `minimumCredits`; `creditsMax` sets a
ceiling on `maximumCredits`. Sending both fences the range on both ends.

## 3.12 Sessions — `sections.sessionCode`

```json
{ "has_child": {
    "type": "enrollmentPackage",
    "query": { "bool": { "should": [
      { "match": { "sections.sessionCode": "YBB" } },
      { "match": { "sections.sessionCode": "YCC" } }
    ]}}
}}
```

Single session collapses to a plain `match` inside `has_child`.

## 3.13 Course ID — `courseId`

```json
{ "term": { "courseId": "019342" } }
```

Zero-padded 6-char string. Used for deep links coming from catalog URLs.

## 3.14 Catalog Number range — `catalogSort`

`catalogSort` is a zero-padded 5-character string. The clause depends on
which bounds are set:

- both bounds, equal → `{"term": {"catalogNumber": "220"}}`
- both bounds, different → `{"range": {"catalogSort": {"gte":"00220","lte":"00325"}}}`
- one bound → `{"range": {"catalogSort": {"gte":"00220"}}}` (or `lte`)

## 3.15 Topic ID — `topics.id`

```json
{ "term": { "topics.id": 12345 } }
```

Used for the "specific topic of a repeatable course" deep link; invisible
in the UI unless reached via URL parameters.

## 3.16 Published (implicit)

When `selectedTerm` is a real term, the client always appends:

```json
{ "has_child": {
    "type": "enrollmentPackage",
    "query": { "match": { "published": true } }
}}
```

This suppresses unreleased sections. Omit it and you may see draft /
pending sections in the result.
