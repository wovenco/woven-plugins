---
name: woven-engagement-points
version: 0.1
description: "Set up, inspect, and manage Woven engagement points, outreach templates, journeys, reengagement check-ins, and live A/B split-test experiments via the Woven External MCP server. Use when the user asks about outreach campaigns, email or SMS templates, Liquid previews, journey graphs or proposals, nudge/snooze/feedback reengagement settings, or split-test experiment results, or when calling search_engagement_points, get_engagement_points, create_engagement_point, update_engagement_point, get_outreach_templates, preview_outreach_template, create_outreach_template, update_outreach_template, get_engagement_point_journey, list_journey_proposals, get_journey_proposal, approve_journey_proposal, get_reengagement_overview, update_reengagement_config, list_experiments, get_experiment_results, conclude_experiment, or list_engagement_point_contacts on the woven-external-mcp server."
---

# Woven Engagement Points

Guidance for agents connected to the **Woven External MCP server** (`woven-external-mcp`). Install this skill alongside that MCP server.

This skill covers **engagement points** (email and SMS outreach campaigns), **outreach templates**, **journeys and proposals**, **reengagement check-ins**, and **live A/B split-test experiments**. Research studies (`search_research_studies`), Knowledge Base playbooks/documents (`woven-knowledge-base`), draft review (`woven-response-review`), and conversation/report analysis (`woven-conversation-insights`) are covered by their own skills.

## MCP tools

### Engagement points

| Tool | Purpose |
|------|---------|
| `search_engagement_points` | Find engagement points by name (`searchQuery`) or list them, optionally filtered by `status` |
| `get_engagement_points` | Settings, pre-launch `setupSteps` checklist, and (with `detail: "full"`) outreach/follow-up email copy for up to 20 IDs |
| `create_engagement_point` | **[WRITE]** Create a new engagement point in review (nothing is sent yet) |
| `update_engagement_point` | **[WRITE]** Edit settings, sender, or email copy — or change `status` (`paused`, `active`, `launch_review`, `deleted`) in a standalone call |
| `list_engagement_point_contacts` | Contacts enrolled in an engagement point (`engagementPointId`) with `journeyStatus` (`pending`, `sent`, `replied`, `skipped`, `error`), message counts, a status `summary`, and a Dashboard contacts link; filter by `journeyStatus` or `search`; `pageSize` up to 50 |

### Outreach templates

| Tool | Purpose |
|------|---------|
| `get_outreach_templates` | Read templates by `engagementPointId` (`outreach` and `followup` slots) or by `templateIds` (up to 20) |
| `preview_outreach_template` | Render a saved `templateId` or draft `content` (`subject`, `html`, `text`) with `sampleData` Liquid variables without saving |
| `create_outreach_template` | **[WRITE]** Create an engagement point's empty `initial` or `followup` Liquid template (`email` or `sms`) |
| `update_outreach_template` | **[WRITE]** Update `subject`, `html`, or `text` on an existing template while keeping its slot and engagement point |

### Journeys & proposals

| Tool | Purpose |
|------|---------|
| `get_engagement_point_journey` | Active journey graph (trigger, delays, split-test branches, follow-ups) plus optional enrolled contacts (`includeContacts`) |
| `list_journey_proposals` | List customer-visible journey proposals (`proposal` pending review or `approved`) for an engagement point |
| `get_journey_proposal` | Full journey proposal by `proposalId`: hypothesis, journey graph (`config`), and approval metadata |
| `approve_journey_proposal` | **[WRITE]** Approve a pending journey proposal (`proposalId`) so Woven can launch or roll it out |

### Reengagement

| Tool | Purpose |
|------|---------|
| `get_reengagement_overview` | Read `view`: `"config"` (settings), `"conversation_plan"` (active plan on one thread), `"running_conversations"`, or `"sent_conversations"` |
| `update_reengagement_config` | **[WRITE]** Configure `nudge`, `snooze`, `feedback`, and `maxNudgeSnoozeMessages` (enabled plans auto-send check-ins when due) |

### Live split-test experiments

| Tool | Purpose |
|------|---------|
| `list_experiments` | List customer-visible A/B experiments (`running`, `completed`, `cancelled`) with contact counts and reply rates |
| `get_experiment_results` | Per-cohort metrics, 14d+ matured cohorts, statistical significance, sample readiness, and `variations[].cohortKey` |
| `conclude_experiment` | **[WRITE]** Pick `winningCohortKey` on a running experiment and roll it out immediately as the active journey |

## Workflows

### 1. Discover, create, or update an engagement point

1. **Search first** — Call `search_engagement_points` before creating or looking up an engagement point.
   - Check `outcome`: when `outcome` is `"no_exact_match"`, no engagement point name contains every word of `searchQuery` and the returned items are fuzzy suggestions — show them as suggestions rather than assuming a match. When `isTerminal` is `true` with `"no_results"`, report that none matched.
2. **Inspect** — Call `get_engagement_points` with `engagementPointIds`. Pass `detail: "full"` whenever the user wants to read or edit email copy (`outreachEmails.outreach` / `outreachEmails.followup` return `subject`, plain-text `body`, and `html`). Check `setupSteps` on pre-launch engagement points to see what remains (`outreach`, `agent_responses`, `audience`).
3. **Create** — Gather facts from the user (never invent goals, audience fields, or URLs) and confirm before calling `create_engagement_point`:
   - `name` and `description` (required)
   - `channel`: `"email"` (default) or `"sms"` (requires SMS access on the account)
   - `contactSource`: `"csv_upload"` (default, email only — contacts are uploaded in the Dashboard) or `"webhook"` (requires `webhookSource` set to the connected platform)
   - Optional: `goal` (`"convert"`, `"learn"`, `"other"`), `goalDescription`, `callToActionType` (`"static"` with an http(s) `callToActionUrl`, `"dynamic"` with a profile field name in `callToActionUrl`, or `"none"` with no URL), `audienceFields` (`{ fieldName, fieldType: "string" | "number" }[]`), `numFollowups` (0–5), `agentTrainingEnabled`
   - Share `nextStep` and the returned Dashboard link so the user can upload contacts or finish setup.
4. **Update settings vs. status** — Always read with `get_engagement_points` first:
   - **Settings / copy call**: Pass `engagementPointId` plus only the fields changing (`name`, `description`, `goal`, `goalDescription`, `callToActionType`, `callToActionUrl`, `numFollowups`, `audienceFields` as the full replacement list, `responseReviewMode` `"manual"` or `"auto"`, `senderName`, `senderEmail` keeping the same `@domain`, `outreachEmail`, `followupEmail`). On a live engagement point, email copy edits apply to new sends immediately.
   - **Standalone status call**: `status` **must** be passed in its own call with **only** `engagementPointId` and `status` — combining `status` with any other field is rejected:
     - `"paused"` (from `active`): stops sending.
     - `"active"` (from `paused`): resumes sending to real contacts.
     - `"launch_review"` (from `external_review`, `training`, or `simulation_review`): approves any pending journey proposal and submits for launch. Trial accounts wait for Woven review; other accounts go live and start sending immediately.
     - `"deleted"` (pre-launch statuses only: `draft`, `internal_review`, `external_review`, `training`, `launch_review`): deletes the engagement point.

### 2. Author, preview, and update outreach templates

1. **Read current slots** — Call `get_outreach_templates` with `engagementPointId` (or `templateIds`, never both) to see whether the `outreach` (`initial`) and `followup` slots exist and inspect their Liquid variables (such as `{{ first_name }}`).
2. **Create an empty slot** — Use `create_outreach_template` only when the target slot (`outreachType`: `"initial"` or `"followup"`) is empty:
   - The `"initial"` template must exist before creating `"followup"`.
   - `channel` must match the engagement point's channel (`"email"` requires both `subject` and `html`, deriving `text` if omitted; `"sms"` requires `text`).
3. **Edit an existing slot** — Use `update_outreach_template` with `templateId` and updated `subject`, `html`, or `text`. Start from the saved `content.html` / `content.text` so layout and Liquid variables stay intact. When `html` is updated without `text`, plain text is regenerated automatically.
4. **Preview Liquid before saving** — Call `preview_outreach_template` with draft `content` (or saved `templateId`, not both) and `sampleData` (for example `{"first_name": "Ada", "planName": "Pro"}`) to verify Liquid syntax and rendered output before calling `create_outreach_template` or `update_outreach_template`.

### 3. Inspect journeys and approve proposals

1. Call `get_engagement_point_journey` with `engagementPointId` to inspect the active journey graph; set `includeContacts: true` (with optional `journeyStatus` or `contactSearch`) when checking contact progression (`pending`, `sent`, `replied`, `skipped`, `error`).
2. Call `list_journey_proposals` with `engagementPointId` to see pending (`proposal`) or `approved` proposals, then `get_journey_proposal` with `proposalId` to read its `hypothesis` and full `config` graph.
3. Show the proposal details and confirm with the user before calling `approve_journey_proposal`.
4. To see who is enrolled, call `list_engagement_point_contacts` (`engagementPointId`, optional `journeyStatus`, `search`, `page`, `pageSize` up to 50) for a status `summary` and each contact's journey status. Use a returned `profileId` with `get_profiles` for profile details, and share the returned Dashboard `links`.

### 4. Configure and monitor reengagement check-ins

1. Read current settings with `get_reengagement_overview` (`view: "config"`, `engagementPointId`). Use `view: "conversation_plan"` (with `convoId` and `channel`) to inspect a single conversation's scheduled plan, or `"running_conversations"` / `"sent_conversations"` (with `cursor` pagination) to audit activity.
2. Before calling `update_reengagement_config`, explain that **enabled plan types (`nudge`, `snooze`, `feedback`) automatically send check-in messages to real contacts when due** (`autoTrigger` locks to `enabled`). Pass the complete config block (including `enabled`) for each plan type being modified (`nudge` supports `maxSteps` 1–5 and `minDelayDays` 1–90; `feedback` supports `delayDays` 1–90; `maxNudgeSnoozeMessages` 1–10 caps combined nudge + snooze messages per contact).

### 5. Review and conclude A/B split-test experiments

1. Call `list_experiments` with `engagementPointId`, then `get_experiment_results` with `experimentId` to inspect per-variation metrics, 14d+ matured cohorts, statistical significance, sample readiness, and `variations[].cohortKey`.
2. Before calling `conclude_experiment`, present the cohort comparison, verify the chosen `winningCohortKey` is a non-holdout variation, explain any holdout traffic split (`winnerTrafficPercentage` 1–100 when a holdout arm is present), and get explicit user confirmation — concluding immediately rolls out the winner to new contacts.

## Presentation rules

- **Dashboard links** — Every tool response includes `links` (each with `label` and `url`). Always include the exact returned `url` values as clickable markdown links when presenting engagement points, setup steps, templates, journeys, reengagement views, or experiments. Never invent or shorten URLs.
- **Before any write tool** — Summarize the exact changes and ask for confirmation. Pay special attention to writes that trigger, alter, or resume real customer outreach:
  - `update_engagement_point` with `status: "active"` (resumes real sends), `status: "launch_review"` (launches / submits for launch), or `status: "deleted"` (removes pre-launch engagement point)
  - `update_engagement_point` or `update_outreach_template` on an **active** engagement point (new sends use updated copy immediately)
  - `update_reengagement_config` when enabling `nudge`, `snooze`, or `feedback` (auto-sends check-ins when due)
  - `approve_journey_proposal` and `conclude_experiment` (activates the winning variation for new contacts)
- **Email/SMS copy** — Show exact `subject` and body text (or rendered preview from `preview_outreach_template`) before saving template or email copy changes.
- **After a write** — Confirm what changed, surface any `notes` or `nextStep` from the response, and share the returned Dashboard link.

## What not to do

- Do not pass `status` alongside other fields in `update_engagement_point` — status transitions must be their own call.
- Do not pass both `engagementPointId` and `templateIds` to `get_outreach_templates`, or both `templateId` and `content` to `preview_outreach_template`.
- Do not call `create_outreach_template` on a slot that already has a template (use `update_outreach_template`), or create `"followup"` before `"initial"`.
- Do not change the domain part of `senderEmail` — only the local part before `@` can be edited.
- Do not select a holdout cohort as `winningCohortKey` in `conclude_experiment`.
- Do not trigger live sends (`status: "active"`, `status: "launch_review"`, enabling reengagement plans, or `conclude_experiment`) without explicit user confirmation.
- Do not invent audience fields, webhook sources, CTA URLs, or Dashboard links.
