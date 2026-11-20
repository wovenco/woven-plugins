# woven-plugins

Official [Woven](https://woven.ai) plugins for Claude and Codex.

The `woven` plugin connects your AI assistant to your Woven account through the Woven MCP server (`https://api.woven.ai/mcp/v1`). Once connected you can review and send AI-drafted replies, explore customer conversations, facets, and reports, and manage engagement points, outreach templates, playbooks, context documents, and research studies. It also includes skills that guide the assistant through each of these workflows.

The Woven plugin is for existing Woven customers. If you don't have a Woven login yet, ask a Woven admin on your team to invite you. The first time the assistant uses a Woven tool, it asks you to sign in with your Woven login. What you can do depends on your role in Woven.

## Install

### Claude Code

```text
/plugin marketplace add wovenco/woven-plugins
/plugin install woven@woven-plugins
```

On Claude Code 2.1.275 or later you can do it in one step:

```text
/plugin install woven --marketplace wovenco/woven-plugins
```

Then run `/mcp`, select `plugin:woven:woven`, and sign in.

### Claude (claude.ai, Claude Desktop, Cowork)

Go to **Customize > Plugins > Add > Add marketplace**, enter `wovenco/woven-plugins`, and install **Woven**. Connect the Woven connector from the plugin's Connectors tab.

### Codex

```text
codex plugin marketplace add wovenco/woven-plugins
codex plugin add woven@woven-plugins
```

In the Codex app, add the marketplace the same way and install Woven from the Plugins directory. Codex asks you to sign in to Woven when the plugin is installed.

## What's included

| Skill | Use it to |
| --- | --- |
| `woven-response-review` | Work through the review queue: read the thread, then approve, edit, or send a reply. |
| `woven-conversation-insights` | Explore conversations, facets, tags, contact profiles, verbatim quotes, and reports. |
| `woven-engagement-points` | Set up and manage engagement points, outreach templates, journeys, reengagement, and split tests. |
| `woven-knowledge-base` | Search and edit playbooks and context documents, see why the AI wrote a reply, and give feedback. |
| `research-study-setup` | Create and edit research studies and interview guides, get test links, and read sessions. |

Sending a reply, resuming outreach, or concluding an experiment reaches real customers, so the assistant asks you to confirm before it runs those tools.

## Support

See the [Woven MCP docs](https://woven.ai/docs/integrations/mcp/) or contact your Woven team.

[Privacy Policy](https://woven.ai/privacy-policy) · [Terms of Service](https://woven.ai/terms-of-service)

## Repository layout

```text
.claude-plugin/marketplace.json   Marketplace catalog (read by Claude and Codex)
plugins/woven/
  .claude-plugin/plugin.json      Claude manifest
  .codex-plugin/plugin.json       Codex manifest and display metadata
  .mcp.json                       Woven MCP server
  skills/                         Workflow skills
```

Bump `version` in both `plugin.json` files when you change a skill or manifest, or installed copies won't update. Tool changes on the Woven MCP server reach users without a new version.
