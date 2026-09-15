# n8n Scheduled API → Transform → Email Report

> A scheduled workflow that pulls data from an API on a timer, reshapes it into a readable HTML table, and emails it out — so a data source gets checked and reported on without anyone having to pull it up manually.

## Problem & Goal

Checking an API endpoint for updates and manually compiling what you find into something readable enough to send someone is a small task that's easy to forget or delay. This workflow automates that loop: on a schedule, it calls an API, extracts the fields that matter, formats them into an HTML table, and emails the result — turning a manual "go check and report" task into something that just runs in the background.

## Architecture

**Schedule Trigger → HTTP Request → Edit Fields → HTML → Gmail**

1. **Schedule Trigger** — kicks off the workflow on a timed interval (the interval itself is left at its default in this version and should be set explicitly before use).
2. **HTTP Request** — calls `GET https://jsonplaceholder.typicode.com/todos/1`. This is a placeholder/test endpoint standing in for the real data source the workflow will eventually point at.
3. **Edit Fields (Set)** — narrows the API response down to two named fields for the report: `title` (string, from `$json.title`) and `userId` (number, from `$json.id`). Note: as configured, `userId` is actually pulled from the todo item's `id` field, not from a `userId` field in the response — worth double-checking once this points at a real API with a different shape.
4. **HTML** — converts the item(s) into an HTML table (`convertToHtmlTable`, with `capitalize` on so field names read as proper headers), producing the report body.
5. **Gmail — Send a message** — sends the email. As configured, the recipient (`ManishPuri484@Gmail.com`), subject (`"Hello"`), and message body (`"New info"`) are all static values — the HTML table produced in the previous step is **not** currently wired into the email's message field, so the generated report isn't actually what gets sent yet.

## Tools & Integrations Used

- **n8n** (Schedule Trigger, HTTP Request, Set/Edit Fields, HTML, Gmail nodes)
- **JSONPlaceholder API** (`/todos/1`) — placeholder data source used while building/testing
- **Gmail API** — OAuth2-connected for sending the report email

## Setup Instructions

1. Import `Project 5-Scheduled API → Transform → Email Report.json` into your n8n instance (Workflows → Import from File).
2. Open the **Schedule Trigger** and set an explicit interval (e.g., daily, hourly) — it's currently unconfigured/default.
3. Replace the **HTTP Request** URL with your real API endpoint, and update the **Edit Fields** node's expressions to match that endpoint's actual response shape.
4. In the **Gmail** node, connect your own **Gmail OAuth2** credentials, and replace the hardcoded `sendTo`, `subject`, and `message` values with real ones — critically, point the `message` field at the **HTML** node's output so the generated table is what actually gets sent.
5. Activate the workflow.

## Product Decisions

- **Separate transform step before formatting:** Pulling the fields of interest out via **Edit Fields** before the **HTML** conversion keeps control over exactly which fields make it into the report, rather than dumping the raw API response into a table.
- **HTML table over plain text:** Formatting the report as an HTML table makes a multi-field record scannable in an inbox, rather than a flat text dump.
- **Placeholder endpoint as a build-time stand-in:** Using JSONPlaceholder during construction lets the shape of the pipeline (fetch → transform → format → send) get built and wired up before the real, likely-authenticated data source is dropped in.

## How I Checked Accuracy

Based on what's actually in the workflow file (rather than claiming a test run that isn't evidenced here), the things worth verifying before treating this as production-ready:

- **Confirm the HTML output is actually used:** right now the Gmail node's `message` field is a static string ("New info"), disconnected from the HTML table the workflow builds — this needs to be wired together and confirmed end-to-end (run it, open the actual email, check the table renders).
- **Confirm `userId` mapping against the real API:** the field is sourced from the response's `id`, not a `userId` field — fine for `todos/1`, but needs re-checking once a real endpoint with a different schema is swapped in.
- **Confirm schedule behavior:** the trigger's interval isn't set explicitly in this version, so the actual run cadence needs to be set and verified rather than assumed.
- **Confirm credentials and recipient:** the Gmail send is hardcoded to one address — needs to be revisited so a test run doesn't email the wrong inbox.

## AI Evaluation

- Reviewed the node graph and connections (Claude) to confirm the workflow is a single linear chain with no branching or error paths, and that every node reference (`$json.title`, `$json.id`) resolves to a field that actually exists in the upstream HTTP response for the current placeholder endpoint.
- Flagged a functional gap: the **HTML** node's table output isn't referenced anywhere in the **Gmail** node's parameters — the email currently sends a static, unrelated message regardless of what the transform step produces. This is the most important fix before relying on this workflow for real reporting.
- Noted the workflow is saved as `"active": false` and still points at a placeholder API and a single hardcoded recipient — consistent with this being a scaffolded/demo version of the pipeline rather than a finished, production-configured one.

## Limitations & Next Steps

- **Report content isn't actually emailed:** the biggest gap — the HTML table needs to be connected into the Gmail message body.
- **Placeholder data source:** `jsonplaceholder.typicode.com/todos/1` needs to be swapped for the real endpoint this is meant to report on.
- **No error handling:** if the HTTP Request fails (timeout, non-200, schema change), there's no fallback — the workflow will just fail silently or error out with no notification.
- **Single hardcoded recipient and static subject/body text:** not yet parameterized for different recipients, environments, or report contents.
- **Single record only:** the HTTP Request targets one fixed item (`todos/1`) rather than a list — extending to a real report likely means fetching and tabulating multiple records, not one.
- **Unconfigured schedule interval:** needs an explicit cadence set before activation.
