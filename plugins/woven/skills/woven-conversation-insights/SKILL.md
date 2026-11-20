---
name: woven-conversation-insights
version: 0.1
description: "Explore customer conversations, semantic conversation facets, contact profiles, validated message quotes, and published reports via the Woven External MCP server. Use when the user asks about customer threads, themes, tones, intents, sentiment, contact lookup, verbatim customer quotes, or campaign and conversation reports, or when calling list_conversations, get_conversations, search_messages, get_message_link, search_conversation_facets, get_conversation_facets, list_conversation_facets, summarize_conversation_facets, get_facet_distributions, list_tags, list_tag_classifiers, search_profiles, get_profiles, list_reports, or get_reports on the woven-external-mcp server."
---

# Woven Conversation Insights, Profiles & Reports

Guidance for agents connected to the **Woven External MCP server** (`woven-external-mcp`). Install this skill alongside that MCP server.

This skill covers **read-only discovery and analysis** across **conversations**, **conversation facets**, **contact profiles**, and **published reports**. Reviewing or sending pending drafts (`woven-response-review`), editing playbooks or documents (`woven-knowledge-base`), managing engagement points (`woven-engagement-points`), and setting up research studies (`research-study-setup`) are covered by their own skills.

## MCP tools

### Conversations & message links

| Tool | Purpose |
|------|---------|
| `list_conversations` | List external conversations filtered **only** by `engagementPointId`, `channel` (`email` or `sms`), `startDate`, and `endDate` |
| `get_conversations` | Full conversation metadata and chronological `messages` for up to 20 `convoIds` on `channel` (`email` or `sms`) |
| `search_messages` | Keyword search over message text (`searchTerm`, `channel`, optional `engagementPointId`, date range, `direction`, `limit` up to 50); returns short snippets, not full bodies |
| `get_message_link` | Validate a verbatim `quote` against a message body and return a direct highlighted Inbox `url` |

### Conversation facets

| Tool | Purpose |
|------|---------|
| `search_conversation_facets` | Hybrid semantic + text search for conversations by meaning, topic, or theme (`query`, `channel`, structured `filters`, date range) |
| `get_conversation_facets` | Fetch extracted facets (`tone`, `intent`, summary, `themes`, entities, etc.) for up to 20 `convoIds` |
| `list_conversation_facets` | Exact-filter, paginated listing of facets (`engagementPointIds`, `tones`, `intents`, `entityTypes`, `entityNames`, dates, `isHighlightConversation`, `isConverted`); `includeContent` adds full facets and `includeAvailableValues` adds valid tone, intent, and entity type values with counts |
| `summarize_conversation_facets` | LLM overview, trends, and highlighted conversations for a filtered set or a semantic `query`; defaults to the last 5 days; limited to 5 calls per minute per user |
| `get_facet_distributions` | Aggregate `{ value, count }[]` frequency counts for `dimension` (`tone`, `intent`, `entityType`, or `entityName`) |

### Contact profiles

| Tool | Purpose |
|------|---------|
| `search_profiles` | Substring search by `searchTerm` across contact name, email, phone number, or explicit `externalId` |
| `get_profiles` | Full profile details (`email`, `phoneNumber`, `emailConsent`, `smsConsent`, `attributes`, `latestContact`) for up to 20 `profileIds`; optional `include` (`activity`, `conversations`, `facets`) adds recent history for up to 10 profiles |

### Tags

| Tool | Purpose |
|------|---------|
| `list_tags` | Customer-visible tags with descriptions and their classifier; filter by `engagementPointId` or `search` (internal tags are never returned) |
| `list_tag_classifiers` | Tag classifiers (themes) with `instructions` and per-tag rules; filter by `engagementPointId` or `isActive` |

### Published reports

| Tool | Purpose |
|------|---------|
| `list_reports` | Discover published external reports (`searchQuery`, `engagementPointIds`, `reportTypes`, `publishedAfter`, `publishedBefore`) with summaries |
| `get_reports` | Retrieve full markdown `content` and `drilldownQuestions` for up to 20 `reportIds` |

## Workflows

### 1. Choose the right search tool (semantic topics vs. exact identifiers vs. date/campaign lists)

 Picking the wrong discovery tool produces misleading answers. Match the tool to the question:

- **Meaning, topics, themes, or sentiment** → `search_conversation_facets`:
  - Pass a conceptual `query` (for example `"complaints about shipping delays"` or `"pricing objections"`), `channel` (`"email"` default or `"sms"`), and optional `filters` (`campaignId`, `campaignIds`, `tone`, `intent`, `sentimentTrajectory`, `isHighlightConversation`, `isConverted`, `failureRiskMin`/`failureRiskMax`, `keywords`, `themes`, `affectedComponents`, `entityType`, `entityName`, `fuzzyMatch`).
  - **Never** call `search_conversation_facets` for an exact ID, reference number, order code, email address, or phone number — not even as a fallback when another lookup returns nothing.
  - Note that `pageInfo.totalCount` on `search_conversation_facets` is approximate (`totalCountIsApproximate: true`).
- **Aggregate trend counts** → `get_facet_distributions`:
  - Call with `dimension` (`"tone"`, `"intent"`, `"entityType"`, or `"entityName"`), `channel`, and optional `engagementPointId` (plus `entityType` filter such as `competitor`, `product`, or `organization` when `dimension` is `"entityName"`, `limit` up to 100) to answer "what are the top intents/tones/competitors?" without paging individual threads.
- **Finding a specific person or contact** → `search_profiles` then `get_profiles`:
  - `search_profiles` runs a substring match against **name, email, phone number, or the profile's explicit `externalId` field only** (ordered by `updated_at` desc, not relevance).
  - `externalId` is **not** a partner ID, member ID, order ID, or arbitrary custom attribute. Do not call `search_profiles` for custom account-specific identifiers unless the user confirms they are stored in `externalId`.
  - Inspect returned fields and require exact equality before confirming an identifier match; call `get_profiles` (`profileIds`, max 20) to read custom `attributes` and consent status.
- **Recent threads for an engagement point, channel, or date window** → `list_conversations` then `get_conversations`:
  - `list_conversations` filters **only** by `engagementPointId` (find via `search_engagement_points` first), `channel` (`"email"` default or `"sms"`), `startDate`, and `endDate`. Do **not** call `list_conversations` hoping an unfiltered page will answer a query about a specific person or custom attribute.
- **Exact filters or a structured listing** → `list_conversation_facets`:
  - Use it when you know the tone, intent, entity, date range, or conversion status to filter on. Set `includeAvailableValues: true` once to see which tone, intent, and entity type values exist (with counts) before filtering. Default results carry only a preview; set `includeContent: true` for full facets.
- **A narrative overview of many conversations** → `summarize_conversation_facets`:
  - Pass the same style of filters, or a conceptual `query` to summarize only the best semantic matches. It runs an LLM, so prefer `get_facet_distributions` or `list_conversation_facets` for counts and plain listings, and do not call it repeatedly (5 per minute per user).
- **Finding messages by the words they contain** → `search_messages`:
  - Use `searchTerm` with optional `engagementPointId`, `startDate`, `endDate`, and `direction`. Results carry snippets only: read the whole thread with `get_conversations` using the returned `convoId`. Email search covers inbound messages and replies, not first outreach messages.
- **How conversations are tagged** → `list_tags` and `list_tag_classifiers`.
- **A contact's history** → `get_profiles` with `include` (`activity`, `conversations`, `facets`).

### 2. Respect `isTerminal` and `recommendedAction` on search outcomes

`list_conversations`, `search_conversation_facets`, and `search_profiles` return structured search-outcome metadata (`outcome`, `isTerminal`, `recommendedAction`):

- When `isTerminal: true` (for example `outcome: "no_results"`), **stop searching that path** and follow `recommendedAction` (report no matches for the applied filters, or ask the user for a supported identifier such as name, email, or phone number). Do not fall back to semantic facet search for an identifier, or dump an unfiltered `list_conversations` page.

### 3. Read threads and validate every cited quote with `get_message_link`

1. Once you have `convoIds` from `list_conversations` or `search_conversation_facets`, call `get_conversations` (up to 20 IDs, specifying `channel`: `"email"` or `"sms"`) to read full message bodies and `messages[].id`, and/or `get_conversation_facets` for structured summaries.
2. **Whenever you quote a specific customer or agent message**, call `get_message_link` with `convoId`, `messageId`, `channelType` (`"email"` or `"sms"`), and `quote`:
   - `get_message_link` verifies that **every word** in `quote` (ignoring punctuation and case) appears as a whole word in that message's actual body, and returns a deep link (`url`) that highlights the message in the Dashboard Inbox.
   - You may join verbatim spans from the same message with `...`, because punctuation is stripped during word validation — never alter, inflect, or paraphrase words inside the quote.
   - If `get_message_link` returns `Quote validation failed` with missing words, re-check the exact text in `get_conversations` and retry with verbatim wording before presenting the quote.

### 4. Discover and read published reports (`list_reports` → `get_reports`)

1. **Discover** — Call `list_reports` first (optional `searchQuery` on title, `engagementPointIds`, `reportTypes`, `publishedAfter`, `publishedBefore`, `page`, `pageSize` default 10 / max 50). Review `title`, `summary`, `typeLabel`, `publishedAt`, and `engagementPoint`.
2. **Read full report** — Call `get_reports` with selected `reportIds` (up to 20 at once) to read the full markdown `content` and `drilldownQuestions`. Missing or unavailable report IDs return `{ id, error: "not_found" }`.
3. Offer relevant `drilldownQuestions` from the report when helpful for follow-up exploration.

## Presentation rules

- **Dashboard links** — Always include clickable markdown links from the `links` array returned by `list_conversations`, `get_conversations`, `search_conversation_facets`, `get_conversation_facets`, `search_profiles`, and `get_profiles` (for example `/inbox?convoId=...` or `/profiles/...`; `summarize_conversation_facets` links its highlighted conversations the same way), and use the `url` from `get_message_link` when citing specific messages. Never invent or construct URLs by hand when a tool returns them, and for reports only share a report URL if one was provided by the user or returned by a tool.
- **Channel clarity** — Email and SMS conversations are separate (`channel: "email"` vs `"sms"`). Always pass the matching `channel` when moving from discovery (`list_conversations`, `search_conversation_facets`) to detail (`get_conversations`, `get_conversation_facets`, `get_message_link`), and label the channel clearly for the user.
- **Simulated conversations** — If `isSimulated` is `true` on a conversation or facet result (for example on demo accounts), note that it is simulated rather than live customer traffic.
- **Approximate counts** — When reporting `totalCount` from `search_conversation_facets`, note that semantic match totals are approximate; use `get_facet_distributions` for exact facet dimension counts.

## What not to do

- Do not pass exact identifiers (emails, phone numbers, order IDs, reference codes) to `search_conversation_facets`.
- Do not pass custom account-specific IDs (order ID, member ID, partner ID) to `search_profiles` unless confirmed to be the profile's `externalId`.
- Do not call `list_conversations` without applicable filters to hunt for a profile or custom attribute.
- Do not ignore `isTerminal: true` by retrying unrelated tools with the same unsupported identifier.
- Do not present direct message quotes without validating them via `get_message_link`, or paraphrase words inside quotation marks.
- Do not exceed batch limits (`20` IDs max for `get_conversations`, `get_conversation_facets`, `get_profiles`, and `get_reports`).
