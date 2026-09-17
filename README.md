# Google Ads MCP Server: Run Your Ad Account From an AI Assistant

A Google Ads MCP server hands an AI assistant typed tools instead of a browser. The assistant
calls `crm_google_ads_breakdown` to pull search terms, `crm_update_google_ads_budget` to move a
daily budget, `crm_set_google_ads_status` to pause a keyword that is burning money. You stay in
the loop for anything that spends, and the work that used to be twenty clicks across six tabs
becomes one sentence.

This repository is the guide to doing that properly: six tutorials, a full tool reference, and
the parts where automation should stop. The worked examples use the hosted MCP server that ships
with [Pinlyx](https://pinlyx.com), which reads and writes **your own** Google Ads account
over OAuth. The shapes translate to any Model Context Protocol server that wraps the Google Ads
API.

```jsonc
// Claude Desktop, Claude Code, Cursor: one block, then restart the client
{
  "mcpServers": {
    "crmsolid-ads": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server", "--tools", "ads"],
      "env": { "CRMSOLID_API_KEY": "csk_live_..." }
    }
  }
}
```

Ask it: *"Which search terms cost money last week and converted nothing?"*

## Start here

| # | Tutorial | What it teaches | Level | Time |
|---|---|---|---|---|
| 01 | [Connect Google Ads to your AI assistant](./tutorials/01-connect-google-ads-to-your-assistant.md) | Create a key, wire the server into Claude Desktop, Claude Code or Cursor, make your first read call | Beginner | 10 min |
| 02 | [Find wasted Google Ads spend](./tutorials/02-find-wasted-google-ads-spend.md) | The search term to negative keyword loop, without burning requests in a loop | Intermediate | 40 min |
| 03 | [Pause campaigns and move budget, safely](./tutorials/03-pause-and-rebudget-safely.md) | The guarded write loop: read, state, confirm, change, verify | Advanced | 45 min |
| 04 | [Publish a campaign with a human approval step](./tutorials/04-publish-a-campaign-with-approval.md) | Build, dry run against Google, approve, publish paused, then enable | Intermediate | 40 min |
| 05 | [A weekly report your assistant can write](./tutorials/05-weekly-google-ads-report.md) | Account, campaign, ad group and search term rollup you can defend in a meeting | Intermediate | 35 min |
| 06 | [Scopes, quotas and safety](./tutorials/06-scopes-quotas-and-safety.md) | Key scopes, read-only sessions, the daily allowance, prompt injection, what to log | Advanced | 50 min |

Read 01 first. The rest assume its vocabulary and its config block.

## What you can do through the tools

Eleven tools cover the account end to end. Full schemas are in
[reference/tools.md](./reference/tools.md).

**Read** (`ads:read`)

| Tool | Answers |
|---|---|
| `crm_list_google_ads_accounts` | Which ad accounts are connected, and what the daily meter reports |
| `crm_google_ads_summary` | Spend, impressions, clicks, CTR, average CPC, conversions, conversion value, ROAS |
| `crm_google_ads_campaigns` | Campaign table with status and spend, most expensive first |
| `crm_google_ads_breakdown` | Ad groups, ads, keywords or search terms, with the same metrics per row |
| `crm_google_ads_campaign_settings` | Live status, channel type, bidding strategy and target, daily budget, networks, schedule |
| `crm_list_ad_drafts` | Campaign drafts waiting to be validated, approved or published |

**Write** (`ads:write`)

| Tool | Changes |
|---|---|
| `crm_set_google_ads_status` | Enable, pause or remove a campaign, ad group, ad or keyword |
| `crm_update_google_ads_budget` | Daily budget of a campaign, in the account currency |
| `crm_update_google_ads_bidding` | Bidding strategy, with optional target CPA or target ROAS |
| `crm_dry_run_ad_draft` | Asks Google to validate a whole campaign without creating anything |
| `crm_publish_ad_draft` | Publishes an approved draft. The campaign is created paused |

## What is deliberately not automated

Ad accounts spend real money, so three things stay in human hands:

1. **Campaign creation is a two-person job.** A draft is built in the panel's campaign builder and
   approved there. The assistant can validate it against Google and publish an approved draft, and
   the result is always created paused. It cannot invent a campaign and put it live.
2. **Enabling spend is explicit.** Publishing never starts delivery. Someone enables the campaign.
3. **Payment and account setup stay in Google.** Adding a payment method, enabling monthly
   invoicing or creating an ad account are not exposed here, and the Google Ads API does not offer
   the first two at all.

There is also no image or video upload and no keyword planner in this tool set today. Where a
workflow needs them, the tutorials say so instead of pretending.

## Requirements

| Requirement | Detail |
|---|---|
| Node.js 20 or newer | For the `npx` bridge |
| An MCP client | Claude Desktop, Claude Code, Cursor, or anything else that speaks the protocol |
| Plan | The MCP server and API keys are on the Business plan. Google Ads in the CRM is on every plan, including Free |
| A Pinlyx account | Holds the ad account connection and the key |

Connecting and using Google Ads inside Pinlyx is available on every plan, including Free, but
the API key and the MCP server are Business-plan developer surfaces, so the MCP path needs
Business. [Sign up](https://app.pinlyx.com/register), connect your Google Ads account under
Insights > Ads > Google Ads, then create an API key at
[settings/developers](https://app.pinlyx.com/settings/developers) with `ads:read`, plus
`ads:write` when you want the assistant to change things.

## Request allowance

MCP access requires Business, and Business has no daily Google Ads cap, so nothing in this guide
is rationed by the meter. The meter is worth understanding anyway: it is what the `dailyQuota`
block reports, the same ad account may be used from the panel on a lower plan, and the caching and
row-count behaviour it describes is what keeps a session cheap.

Google counts API operations per project, so the server meters what reaches Google:

| Plan | Google Ads requests per day, in the CRM panel |
|---|---|
| Free | 50 |
| Pro | 500 |
| Business | Unlimited, and the only plan with MCP access |

Two things keep that further than it sounds. Reports are cached for 15 minutes, so re-reading the
same window costs nothing, and one breakdown call returns up to 500 rows rather than one row per
call. A full weekly report for one account costs about six requests. On a capped plan, past the
limit the API answers HTTP 429 and says when it resets, which is midnight UTC.

## Why an MCP server rather than a chatbot with a Google Ads plugin

The model never sees your credentials. The bridge runs on your machine, holds the API key in its
environment, and exposes tool names and JSON schemas. Every call is executed server side with your
own OAuth grant, scoped to the accounts Google says you may reach. A leaked transcript is not a
leaked account, and a key can be narrowed to `ads:read` or revoked on its own.

The `--read-only` flag goes further: it hides every write tool for that session, so a research
conversation cannot pause a campaign even if the model decides it should.

## Contributing

Corrections are welcome, particularly when a client's behaviour changes. The bar for a tutorial is
that someone reproduced it from a clean machine. See [CONTRIBUTING.md](./CONTRIBUTING.md).

MIT licensed. Pinlyx is a commercial product; this guide is not, and the tutorials name the
places where another server would work just as well.
