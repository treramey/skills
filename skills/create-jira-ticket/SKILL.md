---
name: create-jira-ticket
description: Create a JIRA ticket from user instructions via acli. Uses project from the current branch when possible, lists project epics, recommends the best epic, asks confirmation before creating, uses ADF descriptions, and can attach Figma designs via the Jira integration.
argument-hint: "describe the ticket to create"
---

Create a JIRA ticket from the user's instructions using the **Atlassian CLI (`acli`)**. Prefer the **same project as the current branch** when it can be inferred from a JIRA key in the branch name; otherwise ask which project. Load **epics** for that project, **infer the best epic** for the new work, show a **preview**, and obtain **explicit confirmation** before creating the issue.

**Usage:**
- `/create-jira-ticket` — Pass the ticket description in the same message as `$ARGUMENTS` (summary intent, bug vs feature, acceptance criteria, links, etc.)

**Instructions:**

1. **Validate `acli` is installed and authenticated:**
   - Run `acli auth status`.
   - If the command is not found, display the error below and **STOP**:
     ```
     Atlassian CLI (acli) is not installed.
     Install with: brew tap atlassian/acli && brew install acli
     ```
   - If authentication fails, display the error below and **STOP**:
     ```
     Atlassian CLI is not authenticated.
     Run: acli auth login
     ```

2. **Determine the JIRA project key:**
   - Run `git branch --show-current`. If detached HEAD, note it but continue if a project can still be chosen.
   - Match a JIRA issue key in the branch name with pattern `[A-Z][A-Z0-9]+-[0-9]+` (case-insensitive; normalize to uppercase). If found, set **project key** to the prefix before `-` (e.g. `PROJ-123` → `PROJ`).
   - If no key is found, run:
     ```bash
     acli jira project list --json
     ```
     Use `AskUserQuestion` so the user picks the **project key**.

3. **Parse instructions from `$ARGUMENTS`:**
   - If empty, use `AskUserQuestion` to collect what ticket to create, then continue.
   - Infer where possible:
     - **Summary** (short, imperative title)
     - **Issue type** (Bug, Story, Task, etc.) — from wording ("fix", "broken" → Bug; "add", "implement" → Story/Task)
     - **Description** — draft with clear sections (for bugs: what's wrong, steps, expected vs actual; for stories/tasks: goal and acceptance criteria). You will convert this to **ADF** for JIRA (see **ADF Format Reference** below; JIRA does not render Markdown in the description field).
     - Optional: **labels**, **Figma URLs** (`https://www.figma.com/...`)
     - If the user **names a specific epic** (key like `PROJ-100` or title), treat that as a **manual epic preference** for step 5.
   - **Required fields for status transitions** (must be set at creation so the ticket can later be transitioned — see **Required Fields Reference** below). Resolve each with the listed default; only ask the user if the default is clearly wrong for this ticket:
     - **Customer** → `Silvervine`
     - **PTS and related components** → `SVApi - Silvervine API`
     - **Priority** → `Medium`
     - **Original estimate** → `6h`
     - **Start date** → first day of the **current active sprint** for the project (see step 3a)
     - **Due date** → last day of the **current active sprint** for the project (see step 3a)

3a. **Fetch the current sprint window** (only needed when at least one ticket will be created):
   - List active sprints for the project's boards via REST (acli does not expose sprints directly):
     ```bash
     # Find a board for the project
     curl -s -H "Authorization: Bearer $(acli auth token)" \
       "https://<site>.atlassian.net/rest/agile/1.0/board?projectKeyOrId=<PROJECT-KEY>" | jq '.values[0].id'
     # Then get the active sprint for that board
     curl -s -H "Authorization: Bearer $(acli auth token)" \
       "https://<site>.atlassian.net/rest/agile/1.0/board/<BOARD-ID>/sprint?state=active" \
       | jq '.values[0] | {name, startDate, endDate}'
     ```
   - Use the active sprint's `startDate` → **Start date** and `endDate` → **Due date** (format as `YYYY-MM-DD`).
   - If no active sprint is found, ask the user via `AskUserQuestion` for explicit dates or to skip them. If the project has no board / is not scrum, set both to none and warn the user that status transitions may be blocked until they're filled in.

4. **List epics in the project:**
   - Run:
     ```bash
     acli jira workitem search --jql "project = <PROJECT-KEY> AND issuetype = Epic" --json --limit 50
     ```
     Replace `<PROJECT-KEY>` with the actual key. If the result is empty, try `type = Epic` instead of `issuetype = Epic` only if your Jira version requires it; if still empty, report that no epics were found for this project.
   - From JSON, collect each epic's **key**, **summary**, and **status** (if present) for display.

5. **Recommend an epic:**
   - If the user specified a **manual epic** in step 3 and that key exists in the list (or resolves via `acli jira workitem view <KEY> --fields summary,issuetype --json` as type Epic), use that as **recommended epic**.
   - Else if there are **no epics**, set **recommended epic** to none and skip matching.
   - Else compare the new ticket's **summary + description** (keywords, product area, components) to each epic **summary** (and description if returned). Pick the epic with the strongest thematic match; prefer epics that are **not Done/Closed** when ties exist.
   - Write a **one-sentence rationale** (e.g. "Matches 'Checkout' epic because the work describes payment flow.").

6. **Confirmation preview (mandatory):**
   - Show a clear preview:
     - Project, issue type, summary
     - **Recommended epic** (key + summary + rationale), or "None — no epics in project" / "None — user declined parent"
     - Short description outline or bullet list (not necessarily full ADF in the preview)
     - **Required-for-transition fields:** Customer, PTS and related components, Priority, Original estimate, Start date, Due date (with the resolved sprint name shown next to the dates, e.g. `2026-05-19 → 2026-06-02 (Sprint 47)`)
     - **Figma URLs** if any
   - Use `AskUserQuestion` so the user can:
     - **Create** with the recommended epic (set **parent** to that epic key when creating a child issue — see step 7)
     - **Choose another epic** from the list (show keys + summaries)
     - **Create without epic parent** (omit parent; only if issue type allows — do not force an epic if they choose none)
     - **Edit** — user revises instructions; return to step 3
     - **Cancel** — **STOP**
   - Do **not** call `acli jira workitem create` until the user confirms creation (one of the "create" paths above).

7. **Create the work item with `acli`:**
   - Build an ADF **description** from the composed content (headings, paragraphs, bullet lists, task lists for acceptance criteria). Follow the **ADF Format Reference** below, or run `acli jira workitem create --generate-json` and align with your site's schema.
   - Write a JSON file and run:
     ```bash
     acli jira workitem create --from-json <temp-file.json> --json
     ```
   - Include `"project": "<PROJECT-KEY>"`, `"type": "<IssueType>"`, `"summary": "..."`, and `"description": { ... ADF ... }`.
   - **Required-for-transition fields** (always include — JIRA workflow rejects status moves when these are missing):
     - `"priority": { "name": "Medium" }` (or user override)
     - `"duedate": "<sprint-end YYYY-MM-DD>"`
     - `"timetracking": { "originalEstimate": "6h" }` (or user override; acli/REST also accepts `"originalEstimate": "6h"` at top level depending on schema)
     - **Start date**, **Customer**, and **PTS and related components** are custom fields. Resolve their `customfield_*` IDs once via `acli jira workitem create --generate-json --project <PROJECT-KEY> --type <IssueType>` (or `GET /rest/api/3/field`) and include them in the payload — e.g. `"customfield_10015": "<sprint-start>"`, `"customfield_XXXXX": { "value": "Silvervine" }`, `"customfield_YYYYY": [ { "value": "SVApi - Silvervine API" } ]`. If a custom field ID cannot be resolved, surface that to the user with the field name so they can fill it in manually.
   - **Parent / epic:** If the user confirmed an epic, include `"parent": "<EPIC-KEY>"` **unless** the issue type is **Epic** (do not parent an Epic under another Epic unless the user explicitly asked). If the API rejects `parent` (some classic projects use **Epic Link** custom field instead), read the error, inspect `--generate-json` for your project, and retry with the correct field shape once; if still blocked, tell the user to set the epic in Jira UI and **STOP** after reporting the created key if the issue was created without parent.
   - On success, output the new **key** and **browse URL** from the JSON if present.

8. **Attach Figma designs (if provided):**
   - If instructions included Figma URLs, attach them using the Figma for Jira **Add Design** mechanism.
   - Designs are stored as **issue-level entity properties**. Retrieve the site URL from the created ticket's JSON response, then:
     1. **Determine the Figma property key:** List the issue's entity properties:
        ```bash
        curl -s -H "Authorization: Bearer $(acli auth token)" \
          "https://<site>.atlassian.net/rest/api/3/issue/<JIRA-ID>/properties/"
        ```
     2. **Set the design property:** Property key and value format depend on the Figma for Jira app; typically:
        ```bash
        curl -s -X PUT -H "Authorization: Bearer $(acli auth token)" \
          -H "Content-Type: application/json" \
          "https://<site>.atlassian.net/rest/api/3/issue/<JIRA-ID>/properties/<figma-property-key>" \
          -d '{"figmaDesigns": [{"url": "<figma-url>", "name": "<design-name>"}]}'
        ```
   - If REST calls fail, fall back to putting Figma URLs in the description and notify the user they can use **Add Design** manually.

9. **Edge cases:**
   - Required custom fields: if create fails with validation errors, show fields and ask the user for values, then retry.
   - If `issuetype = Epic` search returns nothing but the project uses a different epic type name, try listing issue types for the project (`acli` / REST) and adjust JQL once.

**Summary output for the user:** project, new issue key, URL, epic used (or none), required-for-transition fields set (with sprint name for the dates), and Figma attachment result (or fallback).

---

## Required Fields Reference

JIRA workflow on Silvervine projects blocks status transitions when any of these fields are empty. Set them at creation so the ticket can be moved later without manual cleanup.

| Field | Default | Notes |
|-------|---------|-------|
| Customer | `Silvervine` | Custom field, single-select. |
| PTS and related components | `SVApi - Silvervine API` | Custom field, multi-select. Override when the work is clearly in a different component. |
| Priority | `Medium` | Standard JIRA field. Override for true Bugs / urgent work. |
| Original estimate | `6h` | Time tracking. Override when scope is obviously larger/smaller. |
| Start date | **first day of the current active sprint** | Fetch via `/rest/agile/1.0/board/<id>/sprint?state=active`. Custom field. |
| Due date | **last day of the current active sprint** | Same source as Start date. Standard `duedate` field. |

If `--generate-json` reveals additional required fields for the project's create screen, surface them to the user in the confirmation preview and either default or ask, then include in the payload.

---

## Status Workflow Reference

Silvervine projects use this workflow. When transitioning a ticket (`acli jira workitem transition <KEY> --status "<TARGET>"`), the target must be reachable from the current status per the table below. The required fields above must all be set or the transition will be rejected.

**Statuses:**
- `Triage` — initial status after creation
- `Refinement`
- `Needs Internal Clarification`
- `Ready for Development`
- `In Progress`
- `In Code Review`
- `In QA`
- `Failed Testing`
- `Ready to Deploy`
- `On Hold`
- `Canceled` (terminal, but reversible to Triage)
- `Done` (terminal)

**Allowed transitions:**

| From | Allowed targets |
|------|-----------------|
| _(create)_ | `Triage` |
| `Triage` | `Refinement`, `Ready for Development`, `Canceled` |
| `Refinement` | `Needs Internal Clarification`, `Ready for Development`, `Canceled` |
| `Needs Internal Clarification` | `Refinement`, `Ready for Development`, `Canceled` |
| `Ready for Development` | `In Progress` |
| `In Progress` | `In Code Review` |
| `In Code Review` | `In Progress`, `In QA` |
| `In QA` | `Failed Testing`, `Ready to Deploy` |
| `Failed Testing` | `In Progress` |
| `Ready to Deploy` | `Done` |
| `Canceled` | `Triage` |
| **Any status** | `On Hold`, `Needs Internal Clarification` |

Notes:
- `On Hold` and `Needs Internal Clarification` are reachable from **any** status — use them as catch-all parking states.
- Before issuing a transition, run `acli jira workitem transitions <KEY> --json` to confirm the target's transition ID for this project (Jira sometimes renames or hides transitions per workflow scheme). If the target is not listed, the ticket is missing a required field — check the Required Fields Reference and back-fill, then retry.

---

## ADF Format Reference

JIRA does **not** render Markdown. Descriptions must use **Atlassian Document Format (ADF)** — a JSON-based document format. The `--description-file` flag treats file content as plain text, so use **`--from-json`** for rich descriptions.

**Important:** The `description` field in the JSON payload must be an ADF object, not a string. Run `acli jira workitem create --generate-json` to see the expected schema for your site.

**Common ADF node types:**

| Markdown | ADF `type` | Notes |
|----------|-----------|-------|
| `## Heading` | `heading` with `attrs.level` | Levels 1–6 |
| Plain text | `paragraph` with `text` children | |
| `**bold**` | `text` with `marks: [{"type": "strong"}]` | |
| `*italic*` | `text` with `marks: [{"type": "em"}]` | |
| `- item` | `bulletList` > `listItem` > `paragraph` | |
| `1. item` | `orderedList` > `listItem` > `paragraph` | |
| `- [ ] task` | `taskList` > `taskItem` (state: `TODO`/`DONE`) | |
| `` `code` `` | `text` with `marks: [{"type": "code"}]` | Inline code |
| Code block | `codeBlock` with `attrs.language` | |
| `---` | `rule` | Horizontal rule |
| `[link](url)` | `text` with `marks: [{"type": "link", "attrs": {"href": "url"}}]` | |

**Example ADF description:**

```json
{
  "version": 1,
  "type": "doc",
  "content": [
    {
      "type": "heading",
      "attrs": { "level": 2 },
      "content": [{ "type": "text", "text": "Description" }]
    },
    {
      "type": "paragraph",
      "content": [{ "type": "text", "text": "What needs to be done and why." }]
    },
    {
      "type": "heading",
      "attrs": { "level": 2 },
      "content": [{ "type": "text", "text": "Acceptance Criteria" }]
    },
    {
      "type": "taskList",
      "attrs": { "localId": "ac-list" },
      "content": [
        {
          "type": "taskItem",
          "attrs": { "localId": "ac-1", "state": "TODO" },
          "content": [{ "type": "text", "text": "First criterion" }]
        },
        {
          "type": "taskItem",
          "attrs": { "localId": "ac-2", "state": "TODO" },
          "content": [{ "type": "text", "text": "Second criterion" }]
        }
      ]
    }
  ]
}
```
