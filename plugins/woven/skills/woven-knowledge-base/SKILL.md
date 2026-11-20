---
name: woven-knowledge-base
version: 0.1
description: "Search, author, validate, and manage Woven Knowledge Base playbooks, context documents, AI response context, guardrails, agent improvements, and reviewer feedback via the Woven MCP server. Use when the user asks about playbooks, company or engagement-point context docs, why the AI wrote a reply or draft, active guardrails, feedback-driven improvements, or submitting thumbs-up/down feedback, or when calling search_playbooks, get_playbooks, check_playbook, create_playbook, update_playbook, delete_playbook, search_documents, get_documents, create_document, update_document, delete_document, get_ai_response_context, get_engagement_point_guardrails, list_agent_improvements, search_feedback, get_feedback, submit_feedback, or update_feedback on the Woven MCP server."
---

# Woven Knowledge Base & AI Context

Guidance for agents connected to the **Woven MCP server** (`https://api.woven.ai/mcp/v1`), which the Woven plugin installs. Your client may show its tools with a prefix; the tool names below are the part after it.

This skill covers **Knowledge Base playbooks**, **Knowledge Base documents**, **AI response context and guardrails**, and **external reviewer feedback**. Approving or sending pending drafts (`woven-response-review`), managing engagement points/templates (`woven-engagement-points`), setting up research studies (`research-study-setup`), and analyzing conversations or reports (`woven-conversation-insights`) are covered by their own skills.

## MCP tools

### Playbooks

| Tool | Purpose |
|------|---------|
| `search_playbooks` | Search and list active playbooks (account-wide or filtered by `engagementPointId`, `searchQuery`, `actionTypes`, and optional `includeRecentUsage`) |
| `get_playbooks` | Full playbook details (`situation`, `guidance`, `requirements`, `exactLanguage`, `actionDefinitions`, `scope`) and optional `includeVersions` for up to 20 IDs |
| `check_playbook` | Validate a draft playbook (`mode: "create"` or `"update"`) against Woven's quality rubric without saving |
| `create_playbook` | **[WRITE]** Create an active playbook after `check_playbook` passes |
| `update_playbook` | **[WRITE]** Update an active playbook and record a version snapshot after `check_playbook` passes |
| `delete_playbook` | **[WRITE]** Delete an active playbook so the AI stops applying it |

### Knowledge Base documents

| Tool | Purpose |
|------|---------|
| `search_documents` | List/search documents in a required `category`: `account_context`, `campaign_context`, `ingested`, or `woven_managed` |
| `get_documents` | Full document content (`latestVersion`), `isContentEditable`, metadata, and optional `includeVersions` for up to 20 IDs |
| `create_document` | **[WRITE]** Create a new `account_context` or `campaign_context` document with a unique `docType` |
| `update_document` | **[WRITE]** Update a document and create a new version (`title`/`content`/`context` on editable docs; `context` only on `ingested`/`woven_managed`) |
| `delete_document` | **[WRITE]** Delete a Knowledge Base document (`company_info` and `style_guide` cannot be deleted) |

### AI response context & guardrails

| Tool | Purpose |
|------|---------|
| `get_ai_response_context` | Inspect the KB `docs`, `playbooks`, `citations`, and `llmReasoningSummary` behind a sent outbound `message` or pending review `draft` |
| `get_engagement_point_guardrails` | Read active `platformEvals`, `accountEvals`, and `campaignEvals` guardrails for an email or SMS engagement point |
| `list_agent_improvements` | Paginated timeline of playbook, document, and guardrail updates made in response to reviewer feedback on an engagement point |

### External reviewer feedback

| Tool | Purpose |
|------|---------|
| `search_feedback` | List human reviewer feedback on an engagement point or research study (`sortBy`: `created_at` or `score`) |
| `get_feedback` | Hydrated feedback items (up to 10 `feedbackIds`) with score, comment, creator, and draft/message/session context |
| `submit_feedback` | **[WRITE]** Submit human rating (`score` 0–1) and optional `comment` on a draft, message, or research session summary |
| `update_feedback` | **[WRITE]** Update `score`, `comment`, or `key` on an existing external human feedback entry |

## Workflows

### 1. Authoring and updating playbooks (`check_playbook` gate)

Playbooks instruct the AI how to respond when a specific customer `situation` arises.

1. **Search and read first** — Call `search_playbooks` (optionally with `engagementPointId` and `includeRecentUsage: true` to see match/reply usage counts) so you don't create duplicates or contradicting rules. Before editing or deleting, call `get_playbooks` with `playbookIds` to inspect the full current content, `engagementPointIds`, and `actionDefinitions`.
2. **Structure content appropriately**:
   - `situation`: specific customer scenario, question, or intent that triggers the playbook.
   - `guidance`: advisory tone, strategy, and approach (default home for most instructions).
   - `requirements`: strict pass/fail compliance or policy rules (use sparingly).
   - `exactLanguage`: verbatim approved wording the AI must reproduce word-for-word (use rarely, only for legal/compliance/brand text).
   - At least one of `guidance`, `requirements`, or `exactLanguage` must be non-empty.
   - Optional `actionDefinitions` (at most one of each):
     - `{"name": "send_to_review", "metadata": {"reviewReason": "..."}}` (`reviewReason` optional)
     - `{"name": "add_ccs", "metadata": {"ccEmails": ["ops@example.com"]}}` (`ccEmails` required, 1–20 emails)
     - `{"name": "create_zendesk_ticket", "metadata": {"subject": "..."}}` (`subject` optional)
   - Scope: omit `engagementPointIds` (or pass `[]`) for an **account-wide** playbook, or pass specific `engagementPointIds` from `search_engagement_points`. When updating `engagementPointIds`, start from the `engagementPointIds` returned by `get_playbooks` plus your additions/removals (`[]` makes it account-wide).
3. **Validate with `check_playbook` until it passes** — Always run `check_playbook` (`mode: "create"` with `name`, or `mode: "update"` with `playbookId`) **before** calling `create_playbook` or `update_playbook`. If `passed` is `false`, revise the draft to address every item in `schemaErrors` and `failingCriteria` and re-run `check_playbook` until `passed: true`.
4. **Confirm and save** — Show the validated playbook fields to the user, get confirmation, and call `create_playbook` or `update_playbook` with a concise `changeSummary` for version history.

### 2. Managing Knowledge Base documents by category

Documents live in four Dashboard categories (`category` parameter on `search_documents`):

| Category | Scope & Purpose | `create_document` | `update_document` | `delete_document` |
|----------|-----------------|-------------------|-------------------|-------------------|
| `account_context` | Account-wide reference docs (`company_info`, `style_guide`, `glossary`, FAQs) | Allowed (omit `engagementPointId`) | `title`, full `content`, `context` | Allowed **except** `company_info` and `style_guide` |
| `campaign_context` | Engagement-point docs (e.g. `agent_context`) | Allowed (requires `engagementPointId`) | `title`, full `content`, `context` | Allowed |
| `ingested` | Crawled webpages (`sourceUrl`) | Not allowed | **`context` only** (`isContentEditable: false`) | Allowed |
| `woven_managed` | Woven-curated reference docs | Not allowed | **`context` only** (`isContentEditable: false`) | Allowed |

1. **Discover and read** — Call `search_documents` with `category` (and optional `engagementPointId` or `searchQuery`), then `get_documents` with `documentIds` to read `latestVersion.content`, `latestVersion.context`, and `isContentEditable` before any edit or delete.
2. **Unique `docType` per scope** — `create_document` requires a `snake_case` `docType` (`/^[A-Za-z0-9_]+$/`) that is unique within the account (`account_context`) or engagement point (`campaign_context`). If a document with that `docType` already exists, use `update_document` instead.
3. **Full content replacement on edits** — When updating `content` on an `account_context` or `campaign_context` document, always pass the **complete** updated document body based on `latestVersion.content` from `get_documents`. On `ingested` and `woven_managed` documents (`isContentEditable: false`), only pass `context` (pass `null` or `""` to clear AI usage notes); passing `title` or `content` is rejected.

### 3. Explaining AI behavior & guardrails

- **Why did the AI write this?** — Call `get_ai_response_context` with `targetType: "message"` (for a sent outbound message ID from `get_conversations`) or `targetType: "draft"` (for a pending draft ID from `list_review_queue` / `get_conversation_review_context`) and `channel` (`"email"` or `"sms"`). Present the matched `playbooks`, `docs`, `citations`, and `llmReasoningSummary`.
- **What rules gate replies?** — Call `get_engagement_point_guardrails` with `engagementPointId` to inspect `platformEvals`, `accountEvals`, and `campaignEvals`.
- **What improved from past feedback?** — Call `list_agent_improvements` with `engagementPointId` to review playbook, document, and guardrail changes linked to reviewer feedback.

### 4. Reading and submitting reviewer feedback

1. **Browse** — Call `search_feedback` with `engagementPointId` (accepts either an engagement point ID from `search_engagement_points` or a research study ID from `search_research_studies`), then `get_feedback` (up to 10 `feedbackIds`) for full context.
2. **Submit or update** — Confirm the target, `score`, and `comment` with the user before calling `submit_feedback` or `update_feedback`:
   - `entityType`: `"draft_message"`, `"draft_sms_message"`, `"email_message"`, `"sms_message"`, or `"research_session"`.
   - `score`: `0` (thumbs down / negative) to `1` (thumbs up / positive). For `"research_session"`, `score` must be `0` or `1` and `key` defaults to `"research_session_summary"` (rating a session summary again replaces its existing summary rating).
   - Negative feedback on drafts and messages triggers Woven's feedback improvement workflow, so include a clear, actionable `comment` describing what should change.

## Presentation rules

- **Knowledge Base & Dashboard links** — Every tool response returns `links` (for example `/knowledge-base/playbooks?kbpsel=...`, `/knowledge-base/account-docs?kbiasel=...`, `/knowledge-base/campaign-docs?kbicsel=...`, `/knowledge-base/ingested-docs?kbesel=...`, `/knowledge-base/woven-docs?kbwsel=...`, Inbox links, or `/engagement-points/{id}/guardrails` and `/improvements`). Always include the exact `url` from `links` as clickable markdown links. Never invent URLs.
- **Before any write tool**:
  - `create_playbook` / `update_playbook`: run `check_playbook` first until `passed: true`, then show the proposed `name`, `situation`, `guidance`/`requirements`/`exactLanguage`, `actionDefinitions`, and scope (`account` vs `engagementPoints`) and get user confirmation.
  - `create_document` / `update_document`: show the proposed `title`, `context`, and changes to `content` (especially before overwriting existing content) and get confirmation.
  - `delete_playbook` / `delete_document`: show the item's name/title and scope, warn that the AI will stop using it immediately, and get explicit confirmation.
  - `submit_feedback` / `update_feedback`: show the target entity, rating (`score`), and `comment` and confirm before submitting.

## What not to do

- Do not call `create_playbook` or `update_playbook` without first passing `check_playbook` (`passed: true`).
- Do not clear all of `guidance`, `requirements`, and `exactLanguage` on a playbook — at least one content section is required.
- Do not send a partial snippet as `content` to `update_document` — read `get_documents` first and send the full updated document body.
- Do not pass `title` or `content` to `update_document` for `ingested` or `woven_managed` documents — only `context` is editable on those categories.
- Do not attempt to delete `company_info` or `style_guide` with `delete_document` — update them with `update_document` instead.
- Do not pass `engagementPointId` when creating an `account_context` document, or omit `engagementPointId` when creating a `campaign_context` document.
- Do not overwrite or delete playbooks or documents without user confirmation.
