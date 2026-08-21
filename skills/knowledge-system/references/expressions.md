# featureExpression and the runtime path

How a set decides it matches, and everything that has to happen around that decision for the
knowledge to reach a document.

---

## 1. The expression tree

`featureExpression` is the only eligibility rule you author. Four node types:

| `type` | Carries | Notes |
|---|---|---|
| `FEATURE` | `slug` | The content-addressed feature slug. Never has children. |
| `AND` | `children` | All children must be true. **Empty `AND` is false.** |
| `OR` | `children` | Any child true. **Empty `OR` is false.** |
| `NOT` | `children` | Only `children[0]` is read; the rest are ignored. |

Nothing else is a node type — a missing or unknown `type` is deleted on write (see "The sanitizer" below).

```yaml
# (Vendor = Acme AND Type = Invoice) OR (Vendor = Globex AND NOT Type = Draft)
featureExpression:
  type: OR
  children:
    - type: AND
      children:
        - { type: FEATURE, slug: vendor-45a62423116103b32d28ee02507f2eeb }
        - { type: FEATURE, slug: document-type-6e49d6c90d404a72f018e53184fa7d5d }
    - type: AND
      children:
        - { type: FEATURE, slug: vendor-fc0427f8993d7dcaa1156510165a4f21 }
        - type: NOT
          children:
            - { type: FEATURE, slug: document-type-827aa3607808acfaa546f6e377922441 }
```

A single-feature set is just:

```yaml
featureExpression:
  type: FEATURE
  slug: vendor-45a62423116103b32d28ee02507f2eeb
```

### Evaluation

Feature slugs are resolved to feature ids **scoped to the set's organization**, then compared with
the features attached to the document family. A `FEATURE` node whose slug resolves to nothing is
simply "feature not present" — false, silently. That is what a mistyped hash looks like from the
outside: a well-formed set that never fires.

---

## 2. The sanitizer deletes nodes on write

The expression is rebuilt every time it is persisted, and invalid nodes are **removed**, not
rejected. This is the second big "my set never matches" cause, because the file you wrote and the
row that got stored differ.

| Input node | Result |
|---|---|
| `FEATURE` with no `slug` | **removed** |
| `FEATURE` with `children` | children **dropped** (a FEATURE never keeps children) |
| unknown or missing `type` | **removed** |
| `NOT` whose children all got removed | **removed** |
| `AND` / `OR` with no surviving children | **kept**, and evaluates **false** |
| whole tree sanitizes to nothing | stored as SQL `NULL` |

Child order is preserved. If a set stops matching after an edit, read the stored expression back and
compare it with the file.

---

## 3. Legacy clauses

Before `featureExpression` existed, matching used clause rows: clauses OR'd, features within a
clause AND'd, each feature positive (must be present) or negative (must be absent); an empty clause
never matches. That form is still evaluated, but **only when `featureExpression` is null**, and it
lives in its own REST resources (`/api/knowledge-clauses`, `/api/knowledge-clause-features`) whose
feature link field is `featureId`.

Do not author clauses, and do not put `clauses:` or `featureUuid:` in a knowledge set file — a
knowledge set has no such field. (`featureUuid` is a project-template sub-schema shape.) A feature's
`uuid` column plays no role in expressions or clause references.

---

## 4. Getting features onto a document family

An expression can only see features that are **attached to the document family**. Nothing in a
knowledge set file attaches them. The ways features get attached:

| How | What |
|---|---|
| `POST /api/knowledge-features/assignToDocumentFamily` | body `{featureId, documentFamilyId}`; requires the `manage-features` permission on the family |
| `DELETE /api/knowledge-features/unassignFromDocumentFamily` | the reverse |
| `POST /api/document-families/{id}/add-knowledge-feature` | family-side equivalent (plus `remove-knowledge-feature`) |
| Intake configuration | an intake can carry feature ids attached to everything it ingests |
| Assessment cascade | features linked to a matching set's **items** (`POST /api/knowledge-features/assignToKnowledgeItem`) are attached to the family when that set applies, and reported back as `addedFeatures` |

Read the current set with `GET /api/knowledge-features/byDocumentFamily/{documentFamilyId}`.

---

## 5. Scoping: which sets are even considered

For a given document family, candidate sets are those in the same organization where:

- `status` is `ACTIVE` (an empty status coalesces to `ACTIVE`),
- the set is not soft-deleted, and
- the set's project is either unset (org-wide) or equal to the project the **caller** supplied.

`priority` is **not** part of this query. Neither is `setType`.

**Where the project comes from is the trap.** It is never derived from the document or its store —
a store can bind to many projects, so store→project is not a function. Only a plan step running
inside a known project passes one. Every other caller — `POST /api/document-families/{id}/assess`,
content upload, store events — passes an empty project and therefore matches **org-wide sets only**.
A project-scoped set is simply invisible on those paths.

Project scoping gotcha: the server does not resolve `projectSlug` to a project id — the CLI does
that for you at push time. A hand-rolled `POST` carrying only `projectSlug` leaves the project id
NULL, and the set quietly becomes org-wide.

---

## 6. Assessment and baking

`POST /api/document-families/{id}/assess` (permission `assess` on the family; org-wide sets only —
see "Scoping" above):

1. Loads the family's attached features and the scoped sets.
2. Evaluates each set — expression first, legacy clauses as fallback — and sorts the outcome into
   new matches, still-matching, snapshot-changed, and no-longer-matching.
3. Writes an applied-knowledge-set row per newly applied set, pinned to that set's
   `currentSnapshotId`.
4. Attaches any cascade features from the set's items.
5. Merges the items into the document.

Response: `applicableKnowledgeSets`, `appliedKnowledgeSets` (each `{knowledgeSetId, name, setType}`),
`addedFeatures` (`{featureId, source}`), plus `totalApplicableKnowledgeSets` and
`totalAppliedKnowledgeSets`.

**Bake order.** Items are written into the document set-by-set in `priority` DESC order, ties broken
by set id ascending, and within a set by `sequenceOrder` then id. Higher-priority items therefore
lead every ordered read. Only items with `active = true` are fetched — an inactive item is never
baked even though it is still present in the set's snapshot.

Because an application is pinned to a snapshot id, editing a set's items after it has been applied
produces a *snapshot-changed* result on the next assessment rather than a silent update; the
applied rows for out-of-date snapshots are listed by
`GET /api/applied-knowledge-sets/knowledge-sets/{id}/outdated`.

---

## 7. Debug checklist for "my set never fires"

1. `GET /api/knowledge-features/byDocumentFamily/{familyId}` — is the feature actually attached?
2. Compare the attached feature's `slug` character-for-character with the `FEATURE` node's `slug`.
   Recompute the hash from the feature's stored `properties` if in doubt.
3. Read the set back and check `featureExpression` survived the sanitizer intact.
4. Check `status` is `ACTIVE` — a candidate set sits at `PENDING_REVIEW` until promoted.
5. Check the set's project. If it carries one, `/assess` and upload events will never match it —
   only a plan step inside that project will — see "Scoping" above.
6. Check the item's `active` flag.
7. Re-run `POST /api/document-families/{id}/assess` and read `applicableKnowledgeSets` — if the set
   is listed there but its items did not appear, the problem is at step 6, not in the expression.
