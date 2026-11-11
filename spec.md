# Supported-Versions Policy Specification

Draft v3

## 1. Purpose and scope

This format lets a package maintainer publish a machine-readable support rule, so that a tool can answer one question: **is version V of package P supported?** The answer is computed from the rule, the list of the package's released versions and an evaluation time. A binding to the Common Lifecycle Enumeration (CLE) lets a CLE document point to such rules and applies its lifecycle events to the answer.

An evaluation returns one of four results:

- **supported**: the rule covers the version.
- **unsupported**: it does not.
- **unknown**: the evidence or the tool is not sufficient to decide, for example an incomplete release list or a versioning scheme the tool does not implement.
- **error**: an input is malformed.

"Supported" means that fixes are expected under a stated commitment. It says nothing about whether the version is vulnerable.

Out of scope: dependency resolution, vulnerability status, entitlements that depend on who is asking, lifetimes of whole release branches (only per-release lifetimes are covered), and reconstructing what was known at a past date.

This is an experimental draft, not an Ecma standard. The `example.org` schema identifier is a placeholder.

## 2. Building blocks

| Building block | Used for |
| --- | --- |
| [PURL (ECMA-427)](https://ecma-tc54.github.io/ECMA-427/) | Identifying the package |
| [VERS (ECMA-428 Annex A)](https://tc54.org/ecma428/#sec-vers), which TC54-TG2 is standardising | Absolute version ranges, used unchanged |
| [CLE (ECMA-428)](https://tc54.org/ecma428/) | Support definitions and lifecycle events |
| This specification | The policy document, the `svers` expression for relative rules, the scheme profiles in Annex A, and the CLE binding |

References of the form "VERS §A.x" point to Annex A of ECMA-428. An `svers` string is an expression, not a URI: it can contain spaces and braces.

## 3. Policy document

```json
{
  "$schema": "https://example.org/svers/policy-1.0.0-draft.schema.json",
  "identifier": "pkg:npm/example",
  "policy": "svers:npm/latest-major:2",
  "prereleases": false,
  "documentation": "https://example.com/support/standard"
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `$schema` | yes | Identifies the document as a policy document and gives the format version |
| `policy` | yes | One `vers:` or `svers:` string, or an array of them. A version is covered if any entry covers it |
| `prereleases` | no | Whether pre-release versions take part in relative (`svers:`) selection. Default `false`. Absolute `vers:` ranges are not affected |
| `identifier` | no | A versionless PURL, or an array of PURL aliases. Present when the document is for one package, absent when several packages share it |
| `documentation` | no | HTTPS URL of the human-readable policy |

No other fields are allowed. A policy document holds only the rule: it has no release list, no dates and no events. It is immutable at its URL; a changed rule is a new document at a new URL.

### 3.1 Package and applicable entries

An evaluation is always for one package, named by a versionless PURL and compared as a canonical PURL string. If the document has an `identifier`, the PURL must be one of its values; otherwise the result is error.

Each `policy` entry names a versioning scheme. An entry applies to the package when its scheme is the one Annex A assigns to the package's PURL type. PURL types with no scheme of their own use the generic schemes (`semver`, `intdot`, `datetime`, `lexicographic`). The constants `vers:all/*` and `vers:none/*` apply to every package.

Entries that do not apply are ignored once their syntax has been checked, so one document can serve several ecosystems. If no entry applies, the result is error. If an entry applies but the tool does not implement its scheme, that entry is unknown; a tool must never substitute another scheme.

## 4. The `svers` expression language

An `svers` expression selects versions relative to what has been released. It resolves to a plain `vers` range once the release list is known.

```text
svers      := "svers:" scheme "/" expression
expression := term ( "|" term )*
term       := selector ( " in " ( term | "{" vers-constraints "}" ) )?
selector   := "all" | "none" | "latest" | "age:" duration
            | counted ":" N ( "+" duration )?
counted    := "latest" | "latest-major" | "latest-minor" | "latest-patch"
N          := positive integer without leading zero
duration   := "P" N ( "D" | "M" | "Y" )
```

- An expression is the union of its terms.
- `selector in filter` first restricts the candidates with the filter, then applies the selector to what is left. Chains are evaluated right to left.
- A filter in braces is a list of VERS constraints for the same scheme, written and interpreted as in a `vers` string. A `|` inside braces belongs to VERS; a `|` outside separates terms. Braces do not nest, and a literal brace inside a version must be percent-encoded.
- Spaces or tabs are required around `in`, allowed around `|` and at both ends, and not allowed anywhere else outside braces.

### 4.1 Selectors

C is the set of candidates: the releases known at the evaluation time, minus pre-releases unless `prereleases` is true.

| Selector | Result |
| --- | --- |
| `all` | C |
| `none` | the empty set |
| `latest`, `latest:N` | the N highest versions in C (N = 1 for `latest`) |
| `latest-major:N` | every candidate in the N highest majors present in C |
| `latest-minor:N` | within each major present in C, every candidate in its N highest minors |
| `latest-patch:N` | within each minor present in C, the N highest versions |
| `age:P` | every candidate released at or before the evaluation time whose lifetime P has not yet expired |

A group with fewer than N members is kept whole. "Major" and "minor" are the grouping keys of the scheme profile (Annex A); a selector that needs a key the scheme does not define is an error.

### 4.2 Lifetimes

`age:P` gives every release its own lifetime, counted from its release instant. A day is 24 hours. For months, add to the calendar month, keep the time of day and clamp the day to the end of the month; a year is 12 months. The release instant is included and the expiry instant is excluded. A release at `2024-02-29T12:00:00Z` with `age:P1Y` expires at `2025-02-28T12:00:00Z`.

A patch release starts its own lifetime; it does not extend the lifetime of its branch.

### 4.3 Grace periods

A counted selector can carry a grace period. `latest-major:1+P1Y` selects the newest major, and each older major until one year after it was superseded. The same suffix works on the other counted selectors, and the rule is the same for all four: the selector counts elements within a scope.

| Selector | Counts | Within |
| --- | --- | --- |
| `latest:N+P` | versions | all candidates |
| `latest-major:N+P` | majors | all candidates |
| `latest-minor:N+P` | minor lines | each major |
| `latest-patch:N+P` | versions | each minor line |

- An element **starts** at the earliest release among its candidates.
- An element is **current** while fewer than N higher elements of its scope have started. Current elements are selected, exactly as without a grace period.
- Otherwise it was **superseded** at the instant when the N-th of its higher elements started. It stays selected until P after that instant, with the calendar arithmetic of section 4.2.

What follows from the rule:

- The clock starts when the successor ships, not when the old line was last released. A patch on the old major is covered during the grace period but does not extend it.
- With `prereleases` false, a pre-release supersedes nothing: the successor starts at its first stable release.
- An element released after it was already superseded, such as a backport, gets what is left of the grace period, counted from the same instant.
- Like `age:`, a grace period needs a dated release list.

Example: the 3.x line was last released as `3.2.0` on 2025-08-01, and `4.0.0` ships on 2026-10-01. The policy `svers:npm/latest-patch:1 in latest-minor:1 in latest-major:1+P1Y` resolves to `vers:npm/3.2.0` before that day, to `vers:npm/3.2.0|4.0.0` from 2026-10-01, and to `vers:npm/4.0.0` from 2027-10-01.

### 4.4 Examples

| Policy in words | Expression |
| --- | --- |
| Only the newest release | `svers:npm/latest` |
| Every release in the two newest majors | `svers:npm/latest-major:2` |
| Newest patch of every minor line in the two newest majors | `svers:npm/latest-patch:1 in latest-major:2` |
| Only the newest release of each of the two newest majors | `svers:npm/latest-patch:1 in latest-minor:1 in latest-major:2` |
| The newest release of the newest major, and of each older major for one year after its successor ships. Three majors can overlap when two majors ship within a year | `svers:npm/latest-patch:1 in latest-minor:1 in latest-major:1+P1Y` |
| The same, but never more than two majors: the older one drops as soon as a third ships | `svers:npm/latest-patch:1 in latest-minor:1 in latest-major:1+P1Y in latest-major:2` |
| The newest major, plus the 2.10 line and later 2.x | `svers:npm/latest-major:1\|all in {>=2.10.0\|<3.0.0-0}` |
| Each release for 18 months | `svers:pypi/age:P18M` |

In SemVer, `<3.0.0-0` excludes the pre-releases of 3.0.0, which `<3.0.0` would include.

## 5. Evaluation

```text
covers(policyDocument, purl, version, releaseList?, asOf?) -> result
```

### 5.1 Inputs

```json
{
  "complete": true,
  "releases": [
    { "version": "4.1.1", "releasedAt": "2026-03-01T00:00:00Z" }
  ]
}
```

- **Release list.** `complete` is the caller's statement that the list is the whole release history of the package. The caller leaves out yanked or withdrawn releases. Either every release has a `releasedAt` or none has. Releases that are equivalent under the scheme count once; if their dates differ, it is an error.
- **Evaluation time.** `asOf` defaults to the current time, captured once. With a dated list, releases after `asOf` are dropped. An undated list describes the present: an explicit `asOf` then makes relative entries unknown, and so does a grace period or `age:`. Timestamps are RFC 3339.
- **What a past or future `asOf` means.** It filters by date only. It does not reconstruct what was known at that time, and it assumes no releases beyond those supplied.

### 5.2 Procedure

1. Validate the policy document, the PURL, the version, the release list and the syntax of every entry. A malformed input is error, whichever entry would have decided the result.
2. Keep the entries that apply to the package (section 3.1). Parse the version, and every version inside those entries, with their scheme.
3. A `vers:` entry is tested directly. It needs no release list.
4. For an `svers:` entry, resolve each term to a set of candidates and take the union. A term that contains `none` is empty and needs no release list. Any other term needs a complete release list; without one it is unknown. With a complete but empty list, every term is empty. Emit the result as `vers:<scheme>/v1|v2|…`, sorted, or as `vers:none/*` when empty, and test the version against it.
5. Combine the entries: error if any entry is an error; otherwise supported if any entry covers the version; otherwise unknown if any entry is unknown; otherwise unsupported.

The result carries the status, the `asOf` that was used and the resolved ranges:

```json
{ "status": "supported", "asOf": "2026-11-01T00:00:00Z", "resolvedRanges": ["vers:npm/4.1.1"] }
```

The same inputs always give the same status and the same ranges. This draft does not standardise reason codes.

### 5.3 How VERS ranges are read

VERS is used as specified, with the scheme profile supplying equality and order. Ecosystem resolver habits, such as npm leaving pre-releases out of ranges, are not applied. These points are spelled out because VERS leaves them open or because its containment pseudocode (§A.4.2), taken literally, gets them wrong:

- A bare version is an equality point. `=` is not a comparator.
- The order of constraints in the input is not significant. They are sorted, then checked: each version appears once under the scheme's equality, and bounds alternate as VERS §A.3 requires. An invalid range is an error.
- A single `>=` or `>` is unbounded above; a single `<` or `<=` is unbounded below.
- A range made only of `!=` constraints contains every version except those.
- `vers:<scheme>/*` contains every version of the scheme, pre-releases included. `vers:all/*` contains everything and `vers:none/*` nothing.
- In emitted ranges, every byte of a version outside `A-Za-z0-9._~-` is percent-encoded.

## 6. CLE binding

A CLE support definition points to its policy document with the existing `url` field. Nothing is added to CLE 1.0.0. A consumer recognises a policy document by its `$schema`; a `url` that leads to anything else, such as a web page, means that definition cannot be evaluated. The human-readable page goes in the policy document's `documentation`.

```text
supported(cle, policyResources, purl, version, releaseList?, asOf?) -> result
```

- `cle` is every page of the CLE document, with the caller's statement that none is missing. Without that statement the result is unknown, because an unseen event could change the answer.
- `policyResources` is what was found at each definition's `url`, fetched beforehand.
- `purl` must be one of the CLE identifiers.

### 6.1 Procedure

1. Apply withdrawals. From the highest event id to the lowest, a `withdrawn` event that has not itself been withdrawn removes its target. A withdrawal is a correction of the record, so it applies whatever `asOf` is.
2. Of the remaining events, keep those whose `effective` time is not after `asOf`.
3. Take the release list from the caller if one is supplied. Otherwise build it from the `released` events, dated by `effective`, and treat it as complete.
4. If an `endOfLife` event has a range that contains the version, the result is unsupported.
5. For each support definition: if an `endOfSupport` event with its `supportId` has a range that contains the version, the definition is unsupported and its policy is not needed. Otherwise evaluate its policy document with `covers`. A definition with no `url`, an unavailable resource or a resource that is not a policy document is unknown.
6. Combine the definitions as in section 5.2. A document with no support definitions gives unsupported.

### 6.2 Rules

- Event ranges are VERS ranges and are read as in section 5.3. A range whose scheme does not apply to the selected PURL is ignored: it is about another ecosystem. If the scheme applies but the tool does not implement it, the tool cannot tell whether the event ends support, and the affected definitions are unknown unless their policy already excludes the version.
- No other event type changes the result. `endOfDevelopment` ends new features, not fixes.
- The policy states the expectation; events are the record. An event always wins over the policy for the versions it names.
- A publisher that links relative policies should publish a `released` event for every release. When the CLE identifiers span several PURL types, the caller must supply the release list for the selected package.
- Support definitions are additive. CLE requires that the meaning of a definition never changes, so a changed rule is a new policy document under a new definition, and the old definition is ended with an `endOfSupport` event (see the open questions for the side effect).
- The result is the position of the publisher of that CLE document. It says nothing about support offered by others.

## 7. Examples and test vectors

### 7.1 Relative policies

Package `pkg:npm/example`, a complete undated release list, no `asOf`:

```text
2.9.0  2.10.0  2.10.1
3.0.0  3.1.0  3.1.1  3.2.0
4.0.0  4.0.1  4.1.0  4.1.1
5.0.0-beta.1
```

| Policy | Resolves to |
| --- | --- |
| `svers:npm/latest` | `vers:npm/4.1.1` |
| `svers:npm/latest-major:2` | `vers:npm/3.0.0\|3.1.0\|3.1.1\|3.2.0\|4.0.0\|4.0.1\|4.1.0\|4.1.1` |
| `svers:npm/latest-patch:1 in latest-major:2` | `vers:npm/3.0.0\|3.1.1\|3.2.0\|4.0.1\|4.1.1` |
| `svers:npm/latest-patch:1 in latest-minor:1 in latest-major:2` | `vers:npm/3.2.0\|4.1.1` |
| `svers:npm/latest in {<4.0.0}` | `vers:npm/3.2.0` |
| `svers:npm/latest-major:1\|all in {>=2.10.0\|<3.0.0-0}` | `vers:npm/2.10.0\|2.10.1\|4.0.0\|4.0.1\|4.1.0\|4.1.1` |
| `svers:npm/latest-major:2` with `prereleases: true` | `vers:npm/4.0.0\|4.0.1\|4.1.0\|4.1.1\|5.0.0-beta.1` |

The last row shows why `prereleases` defaults to false: one beta counts as a major and retires 3.x.

### 7.2 A CLE document with a linked policy

The policy document at `https://example.com/support/security-2.10-v1.json`:

```json
{
  "$schema": "https://example.org/svers/policy-1.0.0-draft.schema.json",
  "identifier": "pkg:npm/example",
  "policy": "vers:npm/>=2.10.0|<2.11.0-0"
}
```

The CLE document:

```json
{
  "$schema": "https://tc54.org/schemas/cle/cle-1.0.0.schema.json",
  "identifier": "pkg:npm/example",
  "updatedAt": "2026-11-01T00:00:00Z",
  "definitions": {
    "support": [{
      "id": "security-2.10",
      "description": "Security fixes for the 2.10 minor line",
      "url": "https://example.com/support/security-2.10-v1.json"
    }]
  },
  "events": [
    {
      "id": 2,
      "type": "endOfSupport",
      "effective": "2026-06-30T00:00:00Z",
      "published": "2026-01-01T00:00:00Z",
      "versions": [{ "range": "vers:npm/>=2.10.0|<2.11.0-0" }],
      "supportId": "security-2.10"
    },
    {
      "id": 1,
      "type": "released",
      "effective": "2024-05-01T00:00:00Z",
      "published": "2024-05-01T00:00:00Z",
      "version": "2.10.1"
    }
  ]
}
```

For version `2.10.1`, the result is supported at `2026-06-29T23:59:59Z` and unsupported from `2026-06-30T00:00:00Z`. From then on it stays unsupported even if the policy document cannot be fetched, because the definition has ended.

### 7.3 Companion files

- `policy-1.0.0-draft.schema.json`: the JSON Schema of the policy document.
- `supported-versions-conformance-draft.json`: 92 cases (74 for `covers`, 18 for `supported`), each with a fixed clock, inputs and the expected status; 38 of them also pin the resolved ranges. They cover selectors, expression syntax, VERS boundaries, lifetimes, grace periods, release-list rules, PyPI versions, which entries apply to a package, withdrawals and event overrides. An implementation needs only the `npm`, `semver` and `pypi` profiles.

## 8. Open questions

- [ ] **Where should scheme profiles live?** Parsing and order belong to VERS, which plans a document per scheme (§A.5). Should Annex A be contributed there, with this specification keeping only the pre-release test and the grouping keys?
- [ ] **Is `url` the right hook in CLE?** CLE describes it as a link to documentation about the policy. A dedicated field, or an inline policy in a later CLE version, would be more explicit.
- [ ] **Integrity of the policy document.** It lives outside the CLE event ledger, so its immutability rests on convention. A digest next to the link, or a transparency log, would close that gap.
- [ ] **Future-dated events.** "Supported until a date" relies on an `endOfSupport` event whose `effective` time is in the future. CLE implies this is allowed but does not say so.
- [ ] **Retiring a support definition.** CLE has no event for it. An `endOfSupport` over all versions works for this binding, but a consumer that ignores policy documents will read it as "everything is unsupported".
- [ ] **Should relative expiry become events?** When a new major pushes an old one out of a `latest-major:2` policy, nothing appears in the ledger. Tools could generate the matching `endOfSupport` events so that plain CLE consumers see the change.
- [ ] **Branch lifetimes.** "Each minor line is supported for 18 months from its own first release" cannot be expressed: a grace period counts from the successor, and `age:` is per release.
- [ ] **Reason codes.** Should the reasons behind unknown and error be standardised so that tools report them alike?
- [ ] **Name.** `svers` is a working name.

## Annex A. Scheme profiles

A profile is what this format needs to know about one versioning scheme:

1. **Parsing and equivalence**: which strings are versions, which spellings denote the same version, and one canonical string for each.
2. **Order**: a total order on versions.
3. **Pre-release test**: whether a version is a pre-release.
4. **Grouping keys** (optional): an ordered major key and minor key.

Items 1 and 2 are what VERS needs for every scheme; until its schemes document exists, each profile names the ecosystem's own definition. Items 3 and 4 are the only things this specification adds. The rest of the specification never looks inside a version string.

A scheme applies to packages of the PURL type shown. A tool states which profiles it implements; an applicable scheme outside that list gives unknown. Where a key is "none", the selectors that need it are errors for that scheme, and `prereleases` has no effect where the pre-release test is "none". A missing number counts as zero.

| Scheme | PURL type | Versions and order | Pre-release | Major key; minor key | Status |
| --- | --- | --- | --- | --- | --- |
| `semver` | types with no scheme of their own | SemVer 2.0.0, strict. Versions that differ only in build metadata are equivalent | has pre-release identifiers | major; major.minor | Defined, in the suite |
| `npm` | `npm` | as `semver` | as `semver` | as `semver` | Defined, in the suite |
| `cargo` | `cargo` | as `semver` | as `semver` | as `semver` | Defined; VERS §A.5 does not list this scheme yet |
| `golang` | `golang` | SemVer 2.0.0 with the leading `v` of Go module versions; `+incompatible` is build metadata | has pre-release identifiers, which includes pseudo-versions | as `semver`. Majors from 2 up are separate modules, so separate PURLs | First draft |
| `pypi` | `pypi` | PEP 440 version scheme. Canonical string: normalised form without trailing zero release segments | has a pre-release or development segment; a post-release alone is not one | (epoch, first release number); the same plus the second number | Defined, in the suite |
| `gem` | `gem` | RubyGems `Gem::Version` | any segment that contains a letter | first number; first two numbers | First draft |
| `maven` | `maven` | Maven `ComparableVersion` | a qualifier that orders before the plain release: alpha, beta, milestone, rc or cr, snapshot | first number; first two numbers | First draft |
| `nuget` | `nuget` | NuGet version normalisation and comparison: SemVer 2.0.0 plus an optional fourth number, labels compared without case | has a pre-release label | first number; first two numbers | First draft |
| `composer` | `composer` | Composer version normalisation and comparison | stability dev, alpha, beta or RC | first number; first two numbers | First draft |
| `deb` | `deb` | Debian version comparison (epoch, upstream version, revision) | none | none | First draft, order only |
| `rpm` | `rpm` | RPM comparison of epoch, version and release | none | none | First draft, order only |
| `alpine` | `apk` | apk-tools version comparison | suffix `_alpha`, `_beta`, `_pre` or `_rc` | none | First draft |
| `intdot` | types with no scheme of their own | VERS §A.5: integers separated by dots | none | first integer; first two integers | Defined |
| `datetime` | types with no scheme of their own | RFC 3339 timestamps | none | none | Defined |
| `lexicographic` | types with no scheme of their own | byte order of the UTF-8 string | none | none | Defined |

"First draft" rows state the intended rule and need review by people who know the ecosystem before they are relied on. The VERS schemes `gentoo`, `cpan` and `generic` have no profile yet. Packages whose PURL type has no scheme of its own, such as `hex`, `github` or `generic`, can already use `semver`, `intdot`, `datetime` or `lexicographic`.
