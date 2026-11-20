---
name: research-study-setup
version: 0.1
description: "Set up and inspect Woven research studies (AI interviews and usability tests) via the Woven External MCP server. Use when the user asks to create, draft, review, or edit a research study, its brief, or its interview guide, to get a test link, to list or read participant sessions, or when calling search_research_studies, get_research_studies, create_research_study, update_research_study_intake, generate_research_study_guide, update_research_study_guide, review_research_study_guide, redraft_research_study_product_brief, create_research_study_test_link, list_research_sessions, or get_research_sessions on the woven-external-mcp server."
---

# Woven Research Study Setup

Guidance for agents connected to the **Woven External MCP server** (`woven-external-mcp`). Install this skill alongside that MCP server.

This skill covers **setting up** a research study (the brief, the guide, review, and testing) and **reading participant sessions**. Launching, inviting participants, and pausing are **not** available over MCP; send the user to the study in Dash for those.

A research study is either an **interview** (a text chat with an AI interviewer) or a **usability** test (a live voice session where participants use a web product and share their screen). Every study has a **brief** (the tools call it `intake`) and, once drafted, a **guide**: the questions, tasks, and screener the AI moderator follows.

## MCP tools

| Tool | Purpose |
|------|---------|
| `search_research_studies` | Find studies by name or ID, or list them all (most recently updated first) |
| `get_research_studies` | Full study: brief, guide (`guide.content`, `guide.versionId`), duration estimate, readiness, shareable link |
| `create_research_study` | **[WRITE]** Create a draft study with its brief. Nothing is sent to anyone |
| `update_research_study_intake` | **[WRITE]** Replace the brief (full brief; omitted optional fields are cleared), optionally rename |
| `generate_research_study_guide` | **[WRITE]** AI-draft the guide from the brief. **Replaces** any existing guide |
| `update_research_study_guide` | **[WRITE]** Save an edited guide (full guide plus `baseVersionId`) |
| `review_research_study_guide` | AI review of the guide's questions and tasks. Changes nothing |
| `redraft_research_study_product_brief` | **[WRITE]** Usability only: redraft the moderator's private product notes on the guide |
| `create_research_study_test_link` | **[WRITE]** Single-use link (valid 2 hours) to take the study as a participant |
| `list_research_sessions` | List participant sessions (most recent first), optionally filtered by `studyId`, `kind`, or `includeTest` |
| `get_research_sessions` | Full session: study context, AI summary, chronological transcript, usability observations and task outcomes, feedback, Dash link |

## Workflow

1. **Find or create**: call `search_research_studies` first so you don't create a duplicate. To start a new study, call `create_research_study` with a `name` and an `intake`:
   - `kind`: `interview` or `usability`
   - `objective`: the decision or question the study investigates
   - Optional: `targetParticipant`, `hypotheses`, `toneAndConstraints`, `screenParticipants`
   - Usability also needs `productUrl` (a full http(s) URL the user gave you), and optionally `usabilityTasks`, `productNotes`, `knowledgeDomain`, `sessionLengthMinutes`
   - Use the user's own words and facts. Never invent details about their business, product, audience, or URLs. Ask when something required is missing.
2. **Draft the guide**: call `generate_research_study_guide` with the `studyId`. It drafts from the saved brief and returns the study with its new guide.
3. **Read**: call `get_research_studies` with `studyIds`. Note `guide.versionId`: every guide edit and review needs it.
4. **Review**: call `review_research_study_guide` with `studyId` and `guideVersionId`. Present the findings and propose fixes.
5. **Edit**: after the user agrees, call `update_research_study_guide` with:
   - `baseVersionId`: the `guide.versionId` you last read
   - `guide`: the **complete** edited guide, starting from `guide.content`. Anything you leave out is removed.
   - `changeSummary`: a short note of what changed and why
   - `previewIntroduction` only to change the participant welcome blurb (`null` clears it)
6. **Test**: call `create_research_study_test_link` and give the user the `url`. Test sessions are excluded from responses.
7. **Launch**: tell the user to open the study in Dash to launch it and invite participants.

### Changing the brief

`update_research_study_intake` replaces the whole brief: start from the study's current `intake` and change only what the user asked. Changing the brief **does not** change an existing guide. After a guide exists, either:

- edit the guide's objective and participant criteria directly with `update_research_study_guide`, or
- regenerate with `generate_research_study_guide`, which replaces the guide (confirm first; see below).

`isBriefChangedSinceDraft` on the study means the brief changed after the guide was drafted. `isGuideStale` means the guide needs attention before testing.

### Usability product brief

For usability studies, the guide carries a **product brief**: the moderator's private notes on the product, drawn from the account's knowledge base, the brief, the product URL, and the tasks. Call `redraft_research_study_product_brief` with `studyId` (and `baseVersionId` from your last read) to refresh it after the tasks or product change. It saves a new guide version; read the study again before any further guide edit. If `isRedrafted` is false, nothing could be drafted and the current product brief was kept.

### Version conflicts

Guide writes are refused when the guide changed since `baseVersionId` (the user may be editing it in Dash at the same time). The tool returns an error with `currentVersionId`. When that happens:

1. Call `get_research_studies` again.
2. Reapply the user's requested changes to the **current** `guide.content`.
3. Retry with the new `guide.versionId` as `baseVersionId`.

Never resend your old guide over the newer one: that would discard the other edits.

### Guide warnings

`generate_research_study_guide` and `update_research_study_guide` may return `guideWarnings`: plan-quality advice (for example a session that runs long). Show them to the user and offer to fix them with `update_research_study_guide`. They don't block saving.

### Question check on edits

`update_research_study_guide` runs the question check on the edit before saving and fixes what it can in the text you changed (a research question nothing covers gets a new section), so the saved guide can differ from what you sent. Tell the user its `guideCheck.summary` in one line: that the check now passes, or what it still flags. `guideCheck` is null when the check couldn't run.

### Unreadable guide

If a study has `isGuideUnreadable: true`, its saved guide can't be read: `guide` is null and every guide write is refused. Tell the user to contact Woven support. **Do not** call `generate_research_study_guide` to replace it.

### Rate limits and plan limits

- Guide drafting and product brief redrafts are rate limited per user. A refused call returns `isRateLimited: true` and `retryAfterSec`. Nothing was changed. Tell the user how long to wait; **do not retry immediately** or in a loop.
- Self-serve plans cap the number of studies and test links per study. Relay the error message as-is and point the user to their Woven account team to raise the limit.
- An error saying research studies aren't available for this account means the feature isn't turned on for them; tell the user to contact their Woven account team.

## Presentation rules

### Study lists (`search_research_studies`)

- One row per study with columns: **Study**, **Kind**, **Status**, **Updated**.
- Format `updatedAt` as a readable date.

### A study's guide (`get_research_studies`)

- Summarize the guide by part: objective, research questions, participant criteria, screener, then the interview sections (`interviewGuide`) or usability tasks (`usabilityTest.tasks`), with the estimated duration (`guide.estimatedDurationMinutes`).
- Show question and task text **verbatim** when the user is reviewing or editing it. Don't paraphrase what participants will see or hear.

### Review findings (`review_research_study_guide`)

- Group findings by question or task, quote the original wording, and propose a rewrite for each.

### Session lists and details (`list_research_sessions`, `get_research_sessions`)

- Use `list_research_sessions` (optionally with `studyId`, `kind`, or `includeTest: true` when inspecting test runs) to find session IDs, then `get_research_sessions` with `sessionIds` to read the AI summary, transcript, usability observations, and task outcomes.
- Link to each session using its `sessionUrl`.

### Before any write tool

- `create_research_study`, `update_research_study_intake`: show the brief you'll save and confirm.
- `generate_research_study_guide` on a study that **already has a guide**: say it will replace the current guide and get explicit confirmation.
- `update_research_study_guide`: show the exact before and after wording of each change and confirm.
- `redraft_research_study_product_brief`: say it replaces the current product brief and confirm.

### After a write

- Confirm what changed, show any `guideWarnings`, and for a test link give the exact `url` and when it expires (`expiresAt`).

## What not to do

- Do not invent brief details, product URLs, participant criteria, or links.
- Do not send a partial guide to `update_research_study_guide`: omitted sections are deleted.
- Do not change, drop, or move a question's or task's `id` when editing the guide: answers and session data reference it. Leave `id` off new questions and tasks; one is assigned on save.
- Choice questions (`type` single_select, multi_select, or scale) have `options` with their own `id`s, which answers reference: keep an option's `id` when editing it, and leave it off new options. Interview chats show the choice questions that lead the guide as answer chips before the open questions; usability tests ask all of them on a page before the voice session. Use one only when a short pick steers the questions after it, and give it an "Other" or "None of these" option when the list could leave someone out.
- Do not reuse a stale `baseVersionId` after a version conflict; read the study again.
- Do not regenerate a guide without confirmation, and never for a study with `isGuideUnreadable: true`.
- Do not retry a rate-limited call before `retryAfterSec` has passed.
- Do not claim you can launch, invite participants, or pause a study: those happen in Dash.
