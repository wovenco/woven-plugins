---
name: woven-response-review
version: 0.1
description: "Review and send AI draft responses via the Woven MCP server. Use when the user asks about the review queue, pending drafts, approving or editing responses, or when calling list_review_queue, get_conversation_review_context, approve_and_send_response_draft, or send_response_message on the Woven MCP server."
---

# Woven Response Review

Guidance for agents connected to the **Woven MCP server** (`https://api.woven.ai/mcp/v1`), which the Woven plugin installs. Your client may show its tools with a prefix; the tool names below are the part after it.

This skill covers **draft review and send** only. Reports and other MCP tools are out of scope.

## MCP tools

| Tool | Purpose |
|------|---------|
| `list_review_queue` | Pending email/SMS drafts awaiting human review (matches Dash Review tab) |
| `get_conversation_review_context` | Full thread, pending draft body, campaign instructions, AI context |
| `approve_and_send_response_draft` | **[WRITE]** Send the Woven draft unchanged |
| `send_response_message` | **[WRITE]** Send an edited version of the draft |

## Workflow

1. **Discover** — Call `list_review_queue`. Optional filter: `campaignId` (engagement point ID).
2. **Drill in** — Pick an item. Call `get_conversation_review_context` with `convoId` and `channel` (`email` or `sms`).
3. **Review** — Read history, `draft.body`, `draft.ccRecipients` (email only), `reviewReason`, and `campaignInstructions`. Use `aiContext` when explaining why the draft was written.
4. **Send or edit** — Only after user confirmation (see Presentation rules):
   - Unchanged: `approve_and_send_response_draft` with `draftId` and `inReplyToMessageId`
   - Edited: `send_response_message` with `sourceDraftId`, `inReplyToMessageId`, and `body` (email CC is inherited from the source draft)
5. **After send** — Share the returned `dashboardLink` to the sent message.

### Stale-message safety

Write tools require `inReplyToMessageId` from `list_review_queue` or `get_conversation_review_context`. It must match the **latest inbound message** in the thread. If the customer replied again since the draft was generated, refresh context and do not send against a stale ID.

### Same conversation, multiple queue items

`list_review_queue` can return several rows for one conversation (multiple pending drafts). `get_conversation_review_context` returns the pending draft for the most recent tagged inbound message. Work through queue items one at a time; re-fetch context after each send.

## Presentation rules

These apply whenever you present MCP results to the user.

### Review queue (`list_review_queue`)

- Show one row per item with columns: **Customer**, **Campaign**, **Reason**, **Last inbound**, **Requested**, **Link**.
- **Customer** — `displayName` on the first line; `customerIdentifier` (email or phone) on a second line. Use `<br>` in the cell.
- **Campaign** — campaign name on the first line; `channel` (`email` or `sms`) on a second line. Use `<br>` in the cell.
- **Last inbound** — format `lastInboundAt` as a readable date (inbound message the draft replies to).
- **Requested** — format `requestedAt` as a readable date (when the draft was sent for review).
- **`previewText` is the inbound message**, not the draft. Do not treat it as the proposed reply.
- **Link** — clickable markdown link from **`dashboardLink`** for every item.
- Do not dump full draft bodies in the list — use `get_conversation_review_context` for draft text.

### Review context (`get_conversation_review_context`)

- Show the **full verbatim** `draft.body`. Do not summarize, paraphrase, or truncate.
- For **email** drafts, show `draft.ccRecipients` when non-empty (these CC addresses are inherited on edited sends).
- Include **`dashboardLink`** to the conversation.
- When quoting thread history, keep customer and agent messages clearly labeled.

### Before any write tool

- Show the **exact message that will be sent** in a fenced code block:
  - Approve as-is: text from `draft.body`
  - Edit: text from the proposed `body` argument
- For **edited email sends**, show `draft.ccRecipients` when non-empty — CC cannot be changed via MCP.
- Ask the user to confirm explicitly. Do not call `approve_and_send_response_draft` or `send_response_message` until they approve.
- If the user wants edits, show the revised full text and confirm again before sending.

### After send

- Confirm success and include the **`dashboardLink`** from the tool response (links to the sent message).

## Example queue summary

```markdown
## Review queue (3)

| Customer | Campaign | Reason | Last inbound | Requested | Link |
|----------|----------|--------|--------------|-----------|------|
| Emily Watson<br>emily.watson@consultancy.biz | First Time Visitor Experience<br>email | High priority trial day prep | May 29, 2026 | Jun 2, 2026 | [Open](http://localhost:5175/inbox?convoId=...) |
```

Use the exact URLs returned by tools — do not invent or shorten them.

## What not to do

- Do not fabricate draft text or dashboard URLs.
- Do not send without showing the exact outbound message and getting confirmation.
- Do not skip `get_conversation_review_context` when the user asks to review a specific draft's content.
- Do not use write tools without a fresh `inReplyToMessageId` from the latest tool response.
- Do not attempt to add, remove, or change CC recipients via MCP — edited sends inherit CC from the source draft only.
