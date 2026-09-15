# Let an AI Pause Campaigns and Move Budget, Safely

Every write in a Google Ads account spends real money or stops it. The guarded write loop is the shape that makes handing those writes to an assistant defensible: read the current settings, state the proposed change in plain language, get a human to confirm it, make one change at a time, then read the account back and prove the change landed.

This tutorial builds that loop with the CRM Solid MCP server against a live Google Ads account. It assumes you have already found something worth changing, for example with [02: Find Wasted Google Ads Spend With an AI Assistant](./02-find-wasted-google-ads-spend.md). The worked example is one decision: a competitor campaign that spent 310 EUR in 30 days without a single conversion gets paused, and its budget moves to the campaign that is converting.

Advanced level. You should already be comfortable with the read tools and with your client's approval prompts.

## What you need

| Requirement | Detail |
|---|---|
| Scopes on the key | `ads:read` and `ads:write`. Reads alone cannot change anything |
| Connected account | Google Ads connected by OAuth in the panel: Insights > Ads > Google Ads |
| Plan allowance | 50 Google Ads requests a day on Free, 500 on Pro, unlimited on Business |
| Client behaviour | A client that asks before each tool call. Approvals are the point of the loop |
| Blast radius | Writes hit a live ad account immediately, and a removal cannot be undone from here |

Run two server entries, not one. The research entry cannot write even if a prompt tells it to:

```jsonc
{
  "mcpServers": {
    "crmsolid-ads-read": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server", "--tools", "ads", "--read-only"],
      "env": { "CRMSOLID_API_KEY": "csk_live_..." }
    },
    "crmsolid-ads-write": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server", "--tools", "ads"],
      "env": { "CRMSOLID_API_KEY": "csk_live_..." }
    }
  }
}
```

The ads family holds eleven tools: six reads and five writes. `--read-only` keeps a tool only when the server marks it `readOnlyHint: true`, so the read entry lists six. The filter is applied twice inside the bridge, once when the tool list is built and again on every call, so a model that saw a write tool name in an earlier turn still cannot call it through the read entry.

Work in the read entry by default. Switch to the write entry for the minutes in which you are actually changing something, and switch back.

## The loop

| Step | What happens | Who decides |
|---|---|---|
| 1 | Read the live settings of everything you are about to touch | Assistant reads, you look |
| 2 | State the change as a before and after diff | Assistant writes it, you read it |
| 3 | Confirm | You, in your own words |
| 4 | One write per turn | Assistant calls, client prompts, you approve |
| 5 | Read the same settings back | Assistant reads, you compare |

Skipping step 1 is the most common mistake, and it is the one that produces the "budget is now 7 EUR instead of 75" class of accident.

## Step 1: Read the settings before you change them

```text
Call crm_google_ads_campaign_settings for customerId 4783920156 and campaignId
20194857363, then again for campaignId 20194857362. Show both as one table:
name, status, channelType, biddingStrategyType, targetCpa, targetRoas,
dailyBudget, budgetCurrency. Do not propose anything yet.
```

```json
{
  "name": "crm_google_ads_campaign_settings",
  "arguments": { "customerId": "4783920156", "campaignId": "20194857363" }
}
```

```json
{
  "id": "20194857363",
  "name": "Search: Competitors",
  "status": "ENABLED",
  "channelType": "SEARCH",
  "biddingStrategyType": "MAXIMIZE_CLICKS",
  "startDate": "2026-03-01 00:00:00",
  "endDate": "",
  "targetGoogleSearch": true,
  "targetSearchNetwork": true,
  "targetContentNetwork": false,
  "dailyBudget": 15.00,
  "budgetCurrency": "EUR",
  "targetCpa": null,
  "targetRoas": null
}
```

`budgetCurrency` is the account currency, and it is the only currency the write tools accept. `endDate` comes back as an empty string when the campaign has no end date, not as `null`.

Note also that `crm_google_ads_campaigns` does not carry a daily budget for Google accounts: `dailyBudget` and `budgetCurrency` are null on those rows. The budget you are about to change is only visible through `crm_google_ads_campaign_settings`, which is exactly why this step exists.

The same read outside any client. The envelope carries the payload as a JSON string inside `content[0].text`, so it is parsed twice:

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer $CRMSOLID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"crm_google_ads_campaign_settings","arguments":{"customerId":"4783920156","campaignId":"20194857363"}}}' \
  | jq -r '.result.content[0].text' | jq .
```

This call is not cached. Every settings read costs one request against the daily allowance, which is the right trade: a cached settings read is exactly the thing you do not want before a write.

**Verify:** the table shows both campaigns, the currency is the one you expect, and the daily budgets match what you see in the Google Ads interface. If a number disagrees with the interface, stop and find out why before writing anything.

## Step 2: State the change, in units a person can check

```text
Propose the change as a table with one row per write you intend to make:
tool | campaign name | field | value now | value after | daily cost change.

Rules:
- Use the values you just read, not values from earlier in this conversation.
- Express every amount in EUR, the account currency, and say so in the row.
- Add a total line: what this does to daily spend across the account.
- Do not call any tool in this turn.
```

A readable diff for the worked example:

| Tool | Campaign | Field | Now | After | Daily change |
|---|---|---|---|---|---|
| `crm_set_google_ads_status` | Search: Competitors | status | ENABLED | paused | up to -15.00 EUR |
| `crm_update_google_ads_budget` | Search: Generic tools | dailyBudget | 60.00 EUR | 75.00 EUR | +15.00 EUR |

Two habits make this table worth the turn. Insisting the assistant re-read the values in step 1 stops it proposing a change against numbers that were true an hour ago. Insisting on the currency name in every row stops the silent unit error described further down.

**Verify:** the total daily change is what you intended, usually zero when you are moving budget rather than adding it. If the two rows do not cancel, you are not moving budget, you are changing the account's spend.

## Step 3: Confirm, in your own words

Type the confirmation yourself. Do not let "yes" be the whole approval, and do not let the assistant restate the plan and treat its own restatement as your consent.

```text
Approved: pause campaign 20194857363 only, and set campaign 20194857362 daily
budget to 75 EUR. Nothing else. One tool call per turn, and report the result
before making the next one.
```

`crm_set_google_ads_status` is annotated as a destructive write, so most clients raise their own confirmation dialog for it. That dialog is a useful second gate and a bad first one: it shows arguments, not intent, and nobody reads `{"level":"campaigns","entityId":"20194857363","status":"paused"}` and notices it is the wrong campaign. The table in step 2 is where a wrong id gets caught.

**Verify:** your confirmation names the entity ids, the new values and the word "only". An approval that says "go ahead" approves whatever the model decided in the meantime.

## Step 4: Make one change per turn

```json
{
  "name": "crm_set_google_ads_status",
  "arguments": {
    "customerId": "4783920156",
    "level": "campaigns",
    "entityId": "20194857363",
    "status": "paused"
  }
}
```

```json
{ "updated": true, "level": "campaigns", "entityId": "20194857363", "status": "paused" }
```

The same call raw, which is the fastest way to prove a write path works without involving a model:

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer $CRMSOLID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"crm_set_google_ads_status","arguments":{"customerId":"4783920156","level":"campaigns","entityId":"20194857363","status":"paused"}}}' \
  | jq -r '.result.content[0].text' | jq .
```

Then the budget, in the next turn:

```json
{
  "name": "crm_update_google_ads_budget",
  "arguments": {
    "customerId": "4783920156",
    "campaignId": "20194857362",
    "dailyBudget": 75
  }
}
```

```json
{ "updated": true, "campaignId": "20194857362", "dailyBudget": 75 }
```

Three things to know about these results.

A write returns a confirmation of what changed, never a data feed. No tool on this server both reads and writes, so a write can never hand the model a fresh pile of numbers to act on unprompted.

`dailyBudget` is a plain number in the account currency, rounded to a whole cent before it reaches Google. Zero or less is refused with `Daily budget must be greater than zero.`

The budget write costs two requests against the daily allowance, not one: the amount lives on a budget resource rather than on the campaign, so the tool looks that resource up and then mutates it. A status change costs one, and so does a bidding change.

**Verify:** each result says `"updated": true` and echoes the id you meant. If the assistant reports success without showing you the result object, ask for the raw result. "Done" is not evidence.

## Step 5: Read it back

```text
Call crm_google_ads_campaign_settings again for both campaigns and show the same
table as before, with a column for what we expected. Flag any row that does not
match.
```

The read-back is not ceremony. Any write invalidates every cached report for that account, so this read goes to Google and returns the live state rather than the 15 minute cached copy of the world as it was before your change. That costs one request per campaign and it is the cheapest insurance in this workflow.

**Verify:** `status` is `PAUSED` on the first campaign and `dailyBudget` is `75.00` on the second. Google reports status in upper case, while the write tool takes it in lower case. That asymmetry is normal and is not a failed write.

## The composite id: `{adGroupId}~{id}`

Campaigns and ad groups are addressed by their own numeric id. Ads and keywords are not. They are addressed by the ad group that contains them, a tilde, and their own id:

| `level` | `entityId` to send | Source fields in a breakdown row |
|---|---|---|
| `campaigns` | `20194857363` | `campaignId` |
| `ad_groups` | `1571940382` | `adGroupId` |
| `ads` | `1571940382~692847113` | `adGroupId` and `id` |
| `keywords` | `1571940382~48213905` | `adGroupId` and `id` |

`crm_google_ads_breakdown` returns the two halves as separate fields, `adGroupId` and `id`, so joining them with a tilde is the caller's job. Get it wrong and the failure is not loud: for `campaigns` and `ad_groups` the server keeps only digits, and for `ads` and `keywords` only digits and the tilde, so a bare ad id survives the cleaning and produces a resource name Google rejects as not found. Put the rule in your prompt:

```text
When you call crm_set_google_ads_status at level ads or keywords, the entityId
must be "{adGroupId}~{id}" built from the same breakdown row. Show me the row
you took both halves from before you call.
```

Search terms have no id of their own at all. In a `search_terms` breakdown, `id` is the search term text, and there is no status tool that accepts it. Excluding a search term is done in the Google Ads interface or the CRM Solid campaign builder, and no MCP tool on this server adds a negative keyword.

**Verify:** before any ad or keyword write, the assistant shows the breakdown row and the id it assembled. One tilde, digits on both sides, nothing else.

## Removing is permanent

`status` takes `enabled`, `paused` and `removed`. The first two are reversible from either side. The third is not: `removed` issues a remove operation against the ad account. It is not a soft delete in CRM Solid, there is no restore tool on this surface, and the entity stops appearing in breakdown reports, which filter removed rows out. Historical spend stays in the account's own reporting, but the thing itself is gone and its settings with it.

There is almost never a reason for an assistant to send `removed`. Pausing stops the spend immediately and keeps the option to switch it back on. Put the ban in the saved prompt:

```text
Never call crm_set_google_ads_status with status "removed". If you believe
something should be removed, say so and stop. I will do it myself in the
Google Ads interface.
```

## What a 429 means in the middle of a workflow

When the daily allowance is spent, the API answers HTTP 429 and says so: `Daily Google Ads limit reached (50 requests). It resets at midnight UTC, or upgrade your plan for more.` Through MCP, the same condition arrives as a tool result with `isError` set to true and that message as its text, not as a transport error.

That distinction matters because an assistant can read a tool error and carry on. In a read session the worst case is a confused summary. In a write session the worst case is a half-applied change set: the pause landed, the budget did not, and the account is now running one converting campaign on a budget that was sized for a different plan.

Three rules follow.

Check the allowance before you start writing. `crm_list_google_ads_accounts` returns `dailyQuota` and costs nothing. A rebudget pass like this one needs about seven requests: two settings reads, one status write, two for the budget write, two settings reads to verify.

Never retry a 429 in a loop. Nothing changes until midnight UTC. A deployment-wide variant exists as well, with the text `Google Ads is busy for today on this plan. Upgrade for uninterrupted access, or try again tomorrow.`

Write down what landed. Ask the assistant for a list of the writes that returned `"updated": true` and the ones that did not, and finish the rest by hand in the Google Ads interface. Leaving a change set half applied overnight is worse than either applying it or not.

## What can go wrong

**The wrong account was selected.** A workspace can have several connected customer ids, and nothing in a pause or a budget call looks wrong when the id belongs to the other account. There is no error to catch: the call succeeds against the wrong account. The guard is the step 1 read, which shows you the campaign name, and a habit of naming the account in the confirmation sentence. An id that is not connected at all fails loudly with `Google Ads account is not connected for this user.`

**The budget was entered in the wrong currency.** `dailyBudget` is a plain number in the account currency. There is no currency argument, no conversion, and no validation beyond "greater than zero". Type 75 into a TRY account intending 75 EUR and the campaign gets roughly a thirtieth of the budget you meant, silently and immediately. Read `budgetCurrency` in step 1 and repeat it in every row of the step 2 table.

**The budget is shared.** The amount lives on a budget resource, and Google lets several campaigns share one. The tool changes the budget attached to the campaign you name, so if that budget is shared, every campaign on it changes with it. Check in the Google Ads interface whether the budget is shared before you move money with this tool.

**You paused the only converting ad group.** Pausing the largest spender is intuitive and sometimes exactly wrong: the biggest spender is often also the biggest converter. Before any pause below campaign level, read `crm_google_ads_breakdown` at `ad_groups` and check conversions, not just cost. A prompt rule that pays for itself: never propose pausing an entity with conversions above zero in the window without saying, in the same sentence, how many conversions it produced.

**A bidding change quietly cleared a target.** `crm_update_google_ads_bidding` sets the strategy you name, with only the target you pass. Switching to `maximize_conversions` without `targetCpa` moves the campaign to that strategy with no target CPA, clearing whatever target it had. The same applies to `maximize_conversion_value` without `targetRoas`, and `maximize_clicks` is sent without a bid ceiling, which clears a previous one. Read the settings first, and pass the target explicitly even when you intend to keep it. `targetRoas` is a ratio, so 3 means 300 percent.

**The assistant reasoned from a cached report.** Reports are cached for 15 minutes. If a colleague changed something eight minutes ago, or the assistant pulled numbers before your last write, the argument for a change can be built on a stale table. Any write busts the cache for that account, so the safe pattern is read, write, read back, and never propose a second change from numbers pulled before the first one.

## Campaign creation is not one of these writes

There is no tool that invents a campaign. Campaigns are built in the CRM Solid panel's Ads Studio, a human approves the draft there, and only then can an assistant publish it:

- `crm_list_ad_drafts` finds the `draftId`, with the draft's status.
- `crm_dry_run_ad_draft` asks the ad network to validate the draft and returns the network's own field-level problems. Nothing is created.
- `crm_publish_ad_draft` publishes a draft that is in status Approved. A draft that has not been approved is refused. What it creates is created paused, so nothing spends until a person enables it with `crm_set_google_ads_status`.

That is the same guarded loop in a different shape: validate, approve, publish inert, enable deliberately.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Write tools are missing from the list | The session is the `--read-only` entry, or the key lacks `ads:write` | Switch to the write entry, then check the key's scopes |
| `Tool 'crm_update_google_ads_budget' requires scope 'ads:write'` | Key has reads only | Grant `ads:write` at [settings/developers](https://app.crmsolid.com/settings/developers) and restart the client |
| `Google Ads account is not connected for this user.` | Wrong `customerId`, or characters other than digits | Re-read it from `crm_list_google_ads_accounts` |
| `Campaign not found in this account.` | The campaign id belongs to another account | Re-read ids from `crm_google_ads_campaigns` for this `customerId` |
| An ad or keyword write fails as not found | `entityId` was a bare id without `{adGroupId}~` | Rebuild it from the breakdown row, both halves, one tilde |
| `Daily budget must be greater than zero.` | Zero, a negative number, or a number sent as a string | Send a positive number in the account currency |
| `Daily Google Ads limit reached` after some writes landed | Allowance spent mid-workflow | Finish the remaining changes by hand, then resume after midnight UTC |
| Status reads `PAUSED` but you sent `paused` | Google reports upper case, the tool takes lower case | Nothing to fix |
| The change does not show in a report | The report was cached before the write, or the report level filters removed rows | Read the settings back rather than a report |

## Exercise

Run the loop end to end on something harmless: take a paused campaign, change its daily budget by one unit of the account currency, verify, then change it back. Count your requests as you go and compare against the seven the worked example needs. You will learn where your client asks for approval, what its dialog shows and what it hides.

Then try the same writes through the `--read-only` entry and watch them fail before any network call, with the message `This session was started with --read-only, which allows only tools that CRM Solid marks as read-only.` That is what a research session should feel like when a prompt wanders.

## Related reading

- [02: Find Wasted Google Ads Spend With an AI Assistant](./02-find-wasted-google-ads-spend.md), the read loop that produces the decisions this one applies
- [Tool reference](../reference/tools.md): every argument, default, scope and annotation on this surface
- [docs.crmsolid.com/integrations/mcp/](https://docs.crmsolid.com/integrations/mcp/)
- [@crmsolid/mcp-server on npm](https://www.npmjs.com/package/@crmsolid/mcp-server)
- [The MCP specification](https://modelcontextprotocol.io)
