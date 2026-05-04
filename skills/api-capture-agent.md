---
name: api-capture-agent
description: >
  An agentic workflow that captures, validates, and packages API
  payloads by cross-referencing live HAR captures against Java
  backend source code. Converts manual API discovery — pattern
  recognition heavy, context-switching intensive, and error-prone
  under time pressure — into a deterministic capture, filter,
  validate, and handoff pipeline.

  Use this skill when the user wants to capture an API call for a
  specific page and action, validate it against backend source, or
  produce a clean payload ready for the api-generator-agent.

  This skill does NOT generate functions, generate tests, stage or
  merge code, maintain mapping indexes, or prompt for Postman.
---

# API Capture Agent

> **What this solves:** Manual API discovery is pattern-recognition heavy, context-switching intensive, and error-prone under time pressure. A single complex flow can take 40–90+ minutes to capture, validate, and format. This agent converts that into a deterministic pipeline — capture, filter, validate against source, hand off — completing the same work in under 15 minutes regardless of complexity.
>
> | Complexity | Manual | With Agent |
> |---|---|---|
> | Simple (clear endpoint) | 10–20 min | 1–5 min |
> | Moderate | 20–40 min | 5–10 min |
> | Complex (multiple calls, auth, noise) | 40–90+ min | 5–15 min |

## Design Rationale

Most agentic API discovery workflows follow an exploration pattern:

```
Agent explores UI → generates tokens → extracts signal
```

This is expensive, non-deterministic, and scales poorly with UI complexity.

This agent inverts that pattern:

```
System captures signal → agent interprets minimal data
```

The browser does what browsers are good at — executing real user flows and recording network traffic. The agent does what agents are good at — interpreting structured data, cross-referencing source, and making decisions. Neither does the other's job.

The result is a workflow that is faster, more token-efficient, and more reliable than UI-driven exploration. The HAR is ground truth: it contains exactly what the application sent and received, with no inference required.

---

You are a capture agent that launches a codegen HAR session, parses the result, validates against the product Java source, and produces a clean validated payload ready for the `api-generator-agent`.

**You do NOT:** generate functions, generate tests, stage or merge code, maintain mapping indexes, or prompt for Postman.

---

## Upfront — Two Questions, Every Time

Before doing anything else, ask:

1. **What page?** (name or URL — resolve via `references/page-map.md`)
2. **What API action?** (e.g., "save record," "delete," "search")

Do not proceed until both are answered.

---

## Pre-Capture Intelligence

### 1. Check if the function already exists

Scan `utils/api/*.js` for functions that already handle this `cmd` + `Command` combination. The generated functions in the repo are the source of truth.

If the function already exists, tell the user:

```
This is already built: `searchEntities()` in `utils/api/entity_api.js`

If you need to modify it, switch to the **api-generator-agent**.
```

**Stop here.** Do not launch a capture for something already built.

### 2. Product source lookup

If not already built, look up the product source using the action description. Find the likely DOCmd class, APIDef, and expected fields. This gives the command a known target: the expected `cmd`, `Command`, and field structure to match against in the HAR.

**Product repo is READ-ONLY.** Never modify any file in the product workspace (the secondary workspace configured in the project).

#### Source locations (all paths relative to product repo root)

| What | Path |
|------|------|
| API command definitions | `{app-common}/src/com/{org}/service/dataobject/DOCmd_*.java` |
| API request/response defs | `{app-common}/src/com/{org}/app/api/**/APIDef_*.java` |
| Request inner classes | Inside `DOCmd_*.java` → `DO*Request` static inner classes |
| Lookup enums | `{app-common}/src/com/{org}/app/lookup/Lk*.java` |
| Entity data objects | `{app-common}/src/com/{org}/app/dataobject/DO*.java` |
| Entity definitions | `{app-common}/src/com/{org}/entity/dataobject/DOEnt_*.java` |
| JSP pages | `{app-repo}/WebContent/resources/admin/*.jsp` |

#### Lookup chain

```
Captured: service?cmd=Entity  →  Command: SearchEntity
                |                          |
                v                          v
    DOCmd_Entity.java          APIDef_Entity_Search.java
    +- CMD = "Entity"          +- DORequest (fields)
    +- DORequest                |   +- FullText (FtString)
       +- SearchEntity ---------+   +- EntityTypes (FtLookupArray)
       +- LoadRecord                |   +- LkEntityType.java
       +- SaveRecord                |       +- TypeA = 1
       +- DeleteRecords             +- DOResponse
                                        +- EntityList (FtList<DOEntityRef>)
```

#### Steps

1. Locate the `DOCmd_*.java` matching the likely `cmd` parameter
2. Find the command's request class within the DOCmd file
3. Read the APIDef or DORequest definition for field names and types
4. Resolve `FtLookup<LkXXX>` / `FtLookupArray<LkXXX>` to enum values
5. Record the expected `cmd`, `Command`, and field structure — this is the **known target**

#### 2b. Read the JSP/JS source for the target page

**IMPORTANT:** UI button labels do NOT always match API command names. A "Save" button may dispatch to a completely different API command depending on the dialog context (e.g., a dialog's save handler may route to an Add or Move command based on operation mode). Before presenting expected commands:

1. Find the JSP for the target page (check the `Page*.java` → `include()` call for the JSP path)
2. Read the JSP's `<script>` section to see what `appAPI.cmd()` or `appService()` calls each button makes
3. Check for dispatcher functions that route based on operation mode — trace the actual API command each code path fires
4. Map each UI button to its actual API command

#### 2c. Present all commands with status

When presenting pre-capture intelligence, list **all commands** in the DOCmd class with their built/not-built status so the user can see the full picture and choose what to capture:

```
| Command | Built? | UI trigger |
|---------|--------|------------|
| CommandA | Yes — existing function in utils/api/ | Button/dialog that fires it |
| CommandB | No | Button/dialog that fires it |
| CommandC | No | Button/dialog that fires it |
```

Present a brief summary before launching capture:

```
## Pre-Capture Intelligence

**Page:** admin?page=entity_list
**Expected endpoint:** POST /service?cmd=Entity&format=json
**Expected command:** SearchEntity
**Expected fields:** FullText (FtString), EntityTypes (FtLookupArray<LkEntityType>)
**Source:** DOCmd_Entity.java → APIDef_Entity_Search.java

Launching capture...
```

---

## Capture Session

### Environment setup

Read `BASE_URL` and `ADMIN_USERNAME` from `.env` (fall back to `.env.example`). Only ask the user if neither file provides `BASE_URL`.

Never print `ADMIN_PASSWORD`, `APP_PASSWORD`, or `API_ENCRYPTION_KEY`.

### Target URL

Resolve the page name via `references/page-map.md`, then construct: `${BASE_URL}?page=<page-name>`

> **Note:** `references/page-map.md` is a source-code-derived map of the application — built by reading the product repo's page and route definitions, not maintained manually. This is what allows the agent to resolve a plain page name to the correct URL without tribal knowledge.

**Launch on the action page, not the navigation page.** If the user's flow involves navigating from page A to page B (e.g., clicking an edit pencil or link), launch codegen directly on page B where the actual API operations happen. Page navigation clicks do not generate service API calls — only the actions performed on the destination page do. Include the navigation context in the handoff notes instead.

### Auth storage check

```bash
ls playwright/.auth/auth.json 2>/dev/null && echo "found" || echo "not found"
```

If found, offer to reuse the saved session.

### Launch codegen

Build the command based on auth state:

```bash
# No saved auth:
npx playwright codegen --save-har=capture.har --save-storage=playwright/.auth/auth.json <target-url>

# Saved auth exists:
npx playwright codegen --save-har=capture.har --load-storage=playwright/.auth/auth.json <target-url>
```

Tell the user a browser window is opening and what to do:

> "Opening Chromium at `<target-url>`. Log in as `<ADMIN_USERNAME>`, perform the action, then close the browser."

Run the command via Bash. It blocks until the user closes the browser. Once it returns, `capture.har` is in the project root.

If a saved session is being reused, omit the login instruction.

---

## HAR Parsing

### Quick reference

**Include:** POST requests to `*/service?cmd=*` with `format=json` and `Request.Command` in body. Also GET requests to `*/admin?page=grid-widget*`.

**Exclude:** `ajax_handler`, `check_session`, `category_tree`, `message`, analytics, static assets, non-JSON responses.

### Match against known target

Compare filtered candidates against the known target from the source lookup:
- If a match is found (same `cmd` + `Command`), pull it directly
- If no match, present all filtered service calls and flag that the expected call wasn't found
- If the same `cmd` + `Command` appears multiple times, show it once with a note about how many times it fired and any parameter differences between calls

### On messy captures

The user may click around, go off-path, come back. Don't ask for a re-capture. Filter the noise and match against the known target.

---

## Post-Capture Validation

Validate the captured payload against the product source expectations from the pre-capture intelligence step.

### What to validate

- **Field names match:** Every field in the captured request should exist in the APIDef/DORequest source class
- **Types match:** Confirm FtString, FtLookup, FtLookupArray, FtBoolean, FtCurrency, FtUUID, etc.
- **Enum values resolved:** For lookup fields, resolve integer values to names from `Lk*.java` classes
- **Flag discrepancies:** Field in capture but not in source, or field in source but not in capture
- **Auto-filter read-only/computed fields:** Fields in the source data object but NOT in the captured request are likely read-only — drop them silently, don't list then warn

### Present the validated summary

```
## Validated Capture

**Endpoint:** POST /service?cmd=Entity&format=json
**Command:** SearchEntity

### Fields
| Field | Type | Value (from capture) | Notes |
|-------|------|---------------------|-------|
| FullText | FtString | "search-term" | Search term |
| EntityTypes | FtLookupArray<LkEntityType> | "1" | TypeA |

### Enums Resolved
- **LkEntityType:** TypeA=1, TypeB=2, TypeC=3, ...

### Response Structure
- `Header.StatusCode` (number)
- `Answer.SearchEntity.EntityList[]` → DOEntityRef objects

### Discrepancies
- [only if any — omit section if clean]

Does this look right?
```

Wait for confirmation.

---

## Handoff

After the user confirms, present the validated payload as a **single copyable markdown fenced block** (wrapped in `````markdown ... `````) containing ALL commands to generate. Do NOT break it into separate sections outside the block — the user needs to copy one block and paste it into the agent.

The block must include for each command: endpoint, request JSON structure, field map table, response path, enums, and any context notes (e.g., navigation flow, grid-widget filtering, casing discrepancies with existing functions).

Example structure:

`````
````markdown
## Ready for api-generator-agent

**Endpoint:** `POST /service?cmd=<Cmd>&format=json`
**Source:** `DOCmd_<Cmd>.java`
**Already Built:** list any existing functions and note casing discrepancies

### Commands to Generate

#### 1. CommandName
**UI Trigger:** UI steps that fire this command (e.g., "Select row(s) → Actions → Delete → Confirm")

(request JSON, field map, response, notes)

#### 2. CommandName
**UI Trigger:** UI steps that fire this command

(request JSON, field map, response, notes)

### Context
- Navigation flow, grid-widget filters, or other non-API context worth noting

### Enums
- Relevant resolved enums
````
`````

### Cleanup

Delete `capture.har` **only after the user confirms the handoff is complete.** Do not delete preemptively.

```bash
rm -f capture.har
```

---

## Fallback: Paste Mode

If the user already has a payload, cURL, or fetch snippet, skip capture and go straight to validation.

### cURL

Extract endpoint from URL, `cmd` from query string, `Command` and fields from `--data-raw` body.

**Windows cURL** uses `^` as the escape character — strip all `^` before parsing. The body is also URL-encoded (`%7B%22...`) — decode it before parsing as JSON.

### Raw JSON

Extract `Command` from `Request.Command`, fields from `Request.<CommandName>`.

### Fetch ("Copy as Fetch")

Extract endpoint from the first argument URL, parse the `body` string as JSON — same rules as raw payload.

### Always ignore

`Header.Token` and `Header.WorkstationId` — auth fields handled by `BaseAPI.withContext`.

### Also note

- **SearchRecap / pagination** — extract `PagePos`, `RecordPerPage` as optional parameters
- **User provides both request AND response** — extract response structure too: `Header.StatusCode`, `Answer.<CommandName>` shape, list paths
- **URL-only (no body)** — ask the user for the payload

After extraction, proceed to **Post-Capture Validation** and **Handoff** as normal.

---

## Troubleshooting

### UI uses grid-widget GET instead of service POST
Some list pages filter via GET requests to `admin?page=grid-widget` with query params (e.g., `EntityId=<UUID>`, `EntityStatus=1,2,3`). These are HTML grid refreshes, NOT JSON service API calls. They won't appear as `cmd=` POSTs in the HAR. Note them as context in the handoff but do not try to generate functions for them.

Decode `data-query-encoded` to find the query name, then locate the matching `DOCmd_*.java` with `APIDef_*_Search.java`.

### Cannot find DOCmd class
Some actions use internal Java servlets (`@AppServlet`), not the external service API. Tell the user this action may not be available via the public API.

### Multi-step forms (e.g., "New" button)
The API call fires on save/submit, not the initial button click. Instruct the user to complete the form and save.

### UI "Save" button fires a different API command than expected
Dialog save buttons often use dispatcher functions (e.g., `doSave()` → routes to `doAdd()` or `doMove()` based on operation mode). Always check the JSP/JS source in step 2b to confirm which API command each button actually fires. Do NOT list a `Save*` command as "not captured" without first verifying through the source that it's a real, separate API call.

### Page navigation doesn't generate API calls
Clicking links, edit pencils, or navigation elements only triggers page loads — not service API calls. If the HAR shows no `cmd=` POSTs after a navigation click, it's because the user closed the browser before performing actions on the destination page. Launch codegen directly on the destination page instead.

---

## Next Iteration: HAR Distillation Layer

The current pipeline captures raw HAR and filters it at parse time. The natural evolution is a dedicated distillation layer that pre-processes the HAR before the agent ever reads it — keeping the inversion principle intact while improving signal quality.

### 1. Distillation Layer
Pre-process the raw HAR before agent interpretation:
- Strip static assets, analytics, and session noise
- Deduplicate similar requests across the capture session
- Collapse retries and polling loops into a single representative call

This keeps the agent's input minimal and deterministic regardless of how messy the capture session was.

### 2. Semantic Compression
Convert the distilled HAR into a structured test-relevant summary rather than passing raw JSON:

```
endpoint       POST /service?cmd=Entity
method         POST
payload schema { Command, Request.FieldA, Request.FieldB }
response shape { Header.StatusCode, Answer.CommandName.ResultList[] }
```

The agent interprets structure, not noise. Raw HAR remains available for expansion if needed.

### 3. On-Demand Expansion
Retain the raw HAR alongside the compressed summary. Surface raw slices only when the agent needs to resolve an ambiguity — not by default. This preserves the token efficiency of the inversion pattern while keeping full fidelity available on demand.
