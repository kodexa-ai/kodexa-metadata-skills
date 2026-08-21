# Validating an ActivityPlan

Two different checkers exist. They do not overlap, and only one of them is reliably on.

| Checker | When | Reach |
|---|---|---|
| **Plan-shape checks** | `POST /api/activity-plans/validate` (design time) and again at activity start | Always on. At start, the first error-severity finding rejects `POST /api/activities` with 400. |
| **Save-time rule set** | on create/update of the plan | Off by default; each deployment can switch it to warn or enforce. Never assume a clean save means a valid plan. |

## Plan-shape checks

Request:

```json
{ "steps": [ ... ], "organizationId": "00000000-0000-4000-8000-000000000001" }
```

Response is always 200:

```json
{ "valid": false,
  "issues": [ { "stepSlug": "post-invoice", "code": "action-edge-undeclared",
                "severity": "error", "message": "..." } ] }
```

`organizationId` is optional but without it CREATE_TASK action edges are skipped, because resolving
them needs the referenced task template. `valid` is false when any issue has `severity: "error"`.

### Errors — the start returns 400 until fixed

| Code | Trigger | Fix |
|---|---|---|
| `per-document-unsupported` | `perDocument: true` on a type other than EXECUTION / LLM / SCRIPT / BRIDGE_CALL | drop the flag, or move the work into a supported type |
| `max-parallel-llm` / `max-parallel-script` / `max-parallel-bridge-call` | `maxParallel` on those types | remove it — those per-document paths are sequential |
| `unknown-dep-slug` | `dependsOn` names a slug no step has | fix the spelling; remember a dep may also name a step's legacy `uuid` |
| `action-edge-undeclared` | `slug:action` where the upstream's live action source has no such action | declare it in `scriptActions`/`promptActions`/`bridgeActions`, or (CREATE_TASK) in the task template's actions |
| `action-edge-upstream-cannot-emit` | action edge off EXECUTION, APPROVAL or AGENT | route from a SCRIPT, LLM, BRIDGE_CALL or CREATE_TASK step instead |
| `any-branch-kind` | `joinPolicy: ANY_BRANCH` on a non-perDocument step, or on CREATE_TASK/AGENT/APPROVAL | make the join a perDocument EXECUTION/LLM/SCRIPT/BRIDGE_CALL step |
| `any-branch-non-routing` | `ANY_BRANCH` in a plan with no routing shape | give the classifier `perDocument: true` **and** declared actions, and point at least one `slug:action` dep at it |
| `any-branch-no-branches` | the join has no await-only deps on perDocument upstreams | write the branch deps as `branch?` |
| `await-no-condition` | a `slug?` dep with no `conditionExpr` (and not a valid ANY_BRANCH join) | add a `conditionExpr`, or make it a plain/action dep |
| `condition-invalid-jsonata` | `conditionExpr` does not compile | fix the expression |
| `condition-hyphen-slug` | `steps.my-step.x` in a condition | backtick it: ``steps.`my-step`.x`` — otherwise JSONata reads the hyphen as subtraction, the reference never resolves, and the condition settles the step NOT_TAKEN |
| `condition-documents-aggregate` | `steps.<slug>.documents[...]` in a perDocument step's condition under routing | that shape does not exist per document; use `steps.<slug>.action` or `.response` |

Some start-time failures have no issue code because they abort the materializer directly:
`step missing required slug or type`, `duplicate step slug "x"`, an action declaring `uuid` and `slug`
with different values, a duplicate action slug within a step, an unresolvable or non-READY
`agentRuntimeRef`, and a `defaultTitleTemplate` that fails to render.

### Warnings — the plan starts, but something is inert or unintended

| Code | Meaning |
|---|---|
| `join-policy-inert` | a non-default `joinPolicy` on a type that does not honour it. The message text is stale — it names only EXECUTION and LLM, but SCRIPT and BRIDGE_CALL persist a join policy too; it is genuinely inert on CREATE_TASK, AGENT and APPROVAL |
| `action-unrouted` | a routing classifier declares an action with no outgoing branch. Documents taking it stop there and are counted in the activity's unrouted total |
| `action-qualifier-on-await-ignored` | `slug?:action` outside an ANY_BRANCH branch dep, where the qualifier is not the thing gating readiness (the await already admits NOT_TAKEN and SKIPPED). Qualify a plain dep, or gate with a `conditionExpr` |
| `action-edge-legacy-uuid` | the edge names a uuid-shaped token the upstream does not currently declare. It is stored verbatim for back-compat and only fires if that exact token is emitted |
| `create-task-set-status-outcome` | `setDocumentStatus` on a CREATE_TASK that also routes on its outcome — the status is written on *any* task outcome. Use SCRIPT steps keyed on the action for outcome-specific status |

## Save-time rule set

Codes are prefixed `activity-plan.`. Cycles and self-dependencies are caught **only** here — the
start-time checker does not detect them, and a cyclic step simply never becomes runnable.

| Code | Meaning |
|---|---|
| `steps.parse-error` | `steps` is not a JSON array of objects |
| `step.slug-required` / `step.slug-format` / `step.slug-unique` | slug missing, not `^[a-z0-9][a-z0-9_-]*$`, or repeated |
| `step.type-required` / `step.type-known` | `type` missing, or not one of `EXECUTION`, `BRIDGE_CALL`, `CREATE_TASK`, `SCRIPT`, `APPROVAL`, `LLM`, `AI_PROMPT`, `AGENT` |
| `depends-on.unknown-step` / `depends-on.self` / `depends-on.cycle` | dangling dep, self-dep, dependency cycle |
| `depends-on.await-requires-condition` | the save-time mirror of `await-no-condition` |
| `condition.invalid-jsonata` | `conditionExpr` does not compile |
| `condition.malformed` | `conditionExpr` is an object with no non-empty `expr` (e.g. `{expression: "..."}`) — it silently defeats the await guard |
| `condition.unknown-step-ref` (warning) | the condition references a `steps.<slug>` that no step provides |
| `bridge.required-fields` | BRIDGE_CALL missing `serviceBridgeRef` or `endpointName` |
| `bridge.unknown-service-bridge` / `bridge.unknown-endpoint` | the referenced bridge or endpoint does not exist |
| `bridge.status-enum` | a literal `status` value in `requestBody` is outside the endpoint's declared enum |
| `bridge.request-source-conflict` | both the `request*` maps and `requestScript` are set |
| `bridge.request-binding-unknown` | a `request*` value references `inputs.<name>` that is not declared in `inputOptions` |
| `input-options.parse-error` / `.name-required` / `.name-unique` / `.type-known` | malformed `inputOptions`, missing/duplicate `name`, or an unaccepted `type` |
| `template.parse-error` | `defaultTitleTemplate` or `defaultDescriptionTemplate` is not valid Go text/template |
| `template.unknown-binding` | a title/description template references `inputs.<name>` not declared in `inputOptions` |
| `too-large` | over 5 MiB of steps JSON, 2000 steps, 500 input options, or a 10 000-character `conditionExpr` |

Note that `template.unknown-binding` and `bridge.request-binding-unknown` are the reason to declare
every input in `inputOptions` even when `inputsSchema` already lists it.

## Suggested loop

1. Write or edit the YAML.
2. `POST /api/activity-plans/validate` with the plan's `steps` and `organizationId`; fix every error
   and read every warning.
3. Push it (`kdx apply -f` or `kdx sync push`).
4. Bind the plan to the project if it is not bound already.
5. Start it once with representative inputs and documents — several failure modes (a bad
   `promptTemplateRef`, an unresolvable `taskStatusSlug`, a script that returns the wrong action) are
   only visible at run time.
