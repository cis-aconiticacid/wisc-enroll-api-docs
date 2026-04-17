# 4. Response Schemas

Field-level documentation for each response payload. Types are given as
TypeScript-ish notation; `?` marks a field that is sometimes absent.

## 1. `CourseHit` — entries in `POST /api/search/v1 → hits[]`

A search hit denormalizes the course document plus enough aggregate
state for the results list. 45 fields were observed.

```ts
interface CourseHit {
  // ── Identity ──
  termCode: string;              // "1264"
  courseId: string;              // "019342"  (zero-padded 6-char)
  subject: Subject;              // see below
  catalogNumber: string;         // "234"     (un-padded)
  catalogSort: string;           // "00234"   (zero-padded, used for sort)

  // ── Presentation ──
  title: string;
  description: string;
  titleSuggest: string;          // search suggestions source

  // ── Credits ──
  minimumCredits: number;
  maximumCredits: number;
  creditRange: string;           // "3" or "3-4"

  // ── Taught-when ──
  firstTaught: string;           // "Fall 1985-1986"
  lastTaught: string;
  typicallyOffered: string;      // "Fall, Spring"
  currentlyTaught: boolean;

  // ── Requirement attributes ──
  coreGeneralEducation: boolean;
  generalEd:       { code: "COM A"|"COM B"|"QR-A"|"QR-B", … }[];
  ethnicStudies?:  { code: "ETHNIC ST" };
  breadths:        { code: "B"|"H"|"L"|"N"|"P"|"S", description: string }[];
  levels:          { code: "E"|"I"|"A", description: string }[];
  foreignLanguage?: { code: "FL1"|…|"FL5" };
  honors?:         { code: "HONORS_ONLY"|"HONORS_LEVEL"|"INSTRUCTOR_APPROVED" };
  lettersAndScienceCredits?: { code: string, description: string };
  sustainability?: { code: string };
  workplaceExperience?: { … };

  // ── Designations & requirements ──
  courseDesignation: string;            // "MATH 234 ‒ CALCULUS"
  courseDesignationRaw: string;
  fullCourseDesignation: string;
  fullCourseDesignationRaw: string;
  courseRequirements?: { /* structured */ };
  advisoryPrerequisites?: string;
  enrollmentPrerequisites?: string;

  // ── Topics & cross-list ──
  approvedForTopics: boolean;
  topics: { id: number, name: string }[];
  allCrossListedSubjects: Subject[];

  // ── Meta ──
  catalogPrintFlag: string;
  academicGroupCode: string;
  gradingBasis: { code: string, description: string };
  repeatable: "Y" | "N";
  gradCourseWork: boolean;
  openToFirstYear: boolean;
  instructorProvidedContent?: { … };
  subjectAggregate: { … };              // pre-joined subject data
  lastUpdated: number;                  // epoch millis

  // ── ES scoring ──
  matched_queries?: string[];           // present when filters carry _name
}
```

### 1.1 `Subject` sub-object

```ts
interface Subject {
  termCode: string;
  subjectCode: string;             // numeric, e.g. "600"
  description: string;             // uppercase, e.g. "MATHEMATICS"
  shortDescription: string;        // UI label, e.g. "MATH"
  formalDescription: string;
  undergraduateCatalogURI: string;
  graduateCatalogURI: string;
  departmentURI: string;
  schoolCollege: {
    academicOrgCode: string;
    academicGroupCode: string;
    schoolCollegeURI: string;
    shortDescription: string;
    formalDescription: string;
  };
  footnotes: string[];
  departmentOwnerAcademicOrgCode: string;
}
```

> Filter values use `subjectCode` (numeric). UI labels use
> `shortDescription`. Details / enrollment package paths also use
> `subjectCode`.

---

## 2. `EnrollmentPackage` — `GET /enrollmentPackages/{t}/{s}/{c} → []`

Each package is one offered configuration of a course in one term. A
4-credit course with two lecture sections and lab variants might have
several packages.

```ts
interface EnrollmentPackage {
  packageEnrollmentStatus: {
    status: "OPEN" | "WAITLISTED" | "CLOSED";
    availableSeats: number;
    waitlistTotal: number;
    currentlyEnrolled: number;
  };
  classMeetings: ClassMeeting[];           // days/times/building/room
  sections: Section[];                     // one per component (LEC, DIS, LAB)
  published: boolean;
  modesOfInstruction: ("Instruction"|"some"|"Only"|"EMPTY")[];
  isAsynchronous: boolean;
  // …
}

interface Section {
  type: "LEC" | "DIS" | "LAB" | "SEM" | … ;
  sectionNumber: string;
  classNumber: number;
  honors: "HONORS_ONLY" | "HONORS_LEVEL" | "INSTRUCTOR_APPROVED" | null;
  comB: boolean;
  sessionCode: string;                     // "YBB", "YCC", …
  classAttributes?: { attributeCode: string, valueCode: string }[];
  instructor?: { name: string, pvi: string, … };
  // …
}
```

The exact shape of `ClassMeeting` and `Section` is large; enumerating
every field is out of scope — inspect one raw payload to see them.

---

## 3. `Details` — `GET /details/{t}/{s}/{c}`

Superset of `CourseHit`. Adds rich catalog text, cross-listed subject
objects (full `Subject` records, not just codes), footnote references,
and all prerequisite trees.

```ts
interface Details extends CourseHit {
  catalogPrintFlag: string;
  crossListedSubjects: Subject[];    // full objects, not just codes
  catalogURIs: { undergraduate?: string, graduate?: string };
  footnotes?: { id: string, text: string }[];
  instructorProvidedContent?: { syllabusURL?: string, … };
  // plus any future additions — schema is not frozen
}
```

The details endpoint sometimes returns a single object and sometimes an
array of one element (a quirk of ES `_source` projection). Normalize
before use.

---

## 4. `Aggregate` — `GET /api/search/v1/aggregate`

Reference data bootstrap. The portal requests this once and caches it.

```ts
interface Aggregate {
  terms: Term[];
  sessions: { termCode: string, sessions: Session[] }[];
  subjects: Record<string /* termCode | "0000" */, Subject[]>;
  specialGroups: SpecialGroup[][];      // list-of-list; outer ordered by term
}

interface Term {
  termCode: string;
  shortDescription: string;     // "SP26"
  longDescription:  string;     // "Spring 2025-2026"
  beginDate:  string;           // ISO
  endDate:    string;
  pastTerm:   boolean;
  // …
}

interface Session {
  sessionCode: string;          // "YBB"
  sessionDescription: string;
  beginDate: string;
  endDate: string;
  // …
}

interface SpecialGroup {
  groupCode: string;
  groupDescription: string;
  // curated collections the portal highlights
}
```

### 4.1 Sentinel term `"0000"`

Key `"0000"` inside `aggregate.subjects` is the **union across all
terms**. Use it when `selectedTerm == "0000"` and you need the subject
filter menu.

---

## 5. Envelope on errors

```jsonc
{
  "found":   0,
  "hits":    [],
  "message": "parsing_exception: ...",
  "success": false
}
```

`success: false` has only been observed when `message` is set. Clients
should treat `success` as authoritative and surface `message` verbatim.
