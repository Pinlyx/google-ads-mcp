# A Weekly Google Ads Report Your Assistant Can Write

A weekly Google Ads report is worth writing only if you can defend every number in it. This tutorial assembles one over MCP in six tool calls: an account summary, a campaign table, the ad groups that earned their spend and the ones that did not, keyword and search term highlights, and a week over week comparison built from two date ranges rather than from memory. It also covers the part most automated reports get wrong, which is what to say when conversion tracking is thin.

Everything here is read-only. If the server is not connected yet, start with [connect Google Ads to your assistant](./01-connect-google-ads-to-your-assistant.md). Two conventions: MCP tool output is camelCase, and a `tools/call` response wraps the payload as a JSON string in `result.content[0].text`, so every JSON block below is that inner payload after one more parse.

## What a weekly report has to answer

Five questions. If a section answers none of them, cut it. A report that opens with impressions is answering a question nobody asked.

1. What did we spend, and did that change?
2. What did the spend produce: clicks, conversions, conversion value?
3. Which campaigns and ad groups carried the result, and which ones only carried cost?
4. Which search terms should become keywords, and which should become negatives?
5. What do we change this week, and what number has to move as a result?

## Set up a session that cannot change anything

Use a separate API key with `ads:read` only, and run the local bridge read-only. A reporting key that can pause a campaign will eventually pause a campaign.

```jsonc
{
  "mcpServers": {
    "crmsolid-ads-report": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server", "--read-only", "--tools", "ads"],
      "env": { "CRMSOLID_API_KEY": "csk_live_readonly_key" }
    }
  }
}
```

`--read-only` drops every write tool in the proxy before your client sees the list, so `crm_set_google_ads_status` and `crm_update_google_ads_budget` are not merely refused, they are not listed. `--tools ads` narrows the surface to the Google Ads family. Both filters run locally, and a filtered tool is not callable at all. Get the `customerId` first, because every other tool needs it.

```bash
export CRMSOLID_API_KEY="csk_live_readonly_key"
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer $CRMSOLID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"crm_list_google_ads_accounts","arguments":{}}}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).result.content[0].text))"
```

```json
{
  "count": 1,
  "accounts": [
    { "customerId": "5093114872", "name": "CRM Solid TR", "currency": "TRY", "timeZone": "Europe/Istanbul",
      "status": "Connected", "connectedAt": "2026-04-02T09:14:00Z", "lastSyncedAt": "2026-09-15T07:02:11Z" }
  ],
  "dailyQuota": { "used": 0, "limit": null, "remaining": null, "unlimited": true }
}
```

**Verify:** you have a digits-only `customerId`, and `dailyQuota` reads `unlimited: true`, which is what Business returns. That call reads the CRM database and never reaches Google, so it costs nothing and you can run it as often as you like.

## The six calls, and what the report costs

Only requests that actually reach Google count. MCP access requires Business, and Business has no daily Google Ads cap, so the counts below are the price of the report rather than a ration. They are worth knowing anyway: they are what the `dailyQuota` block meters for anyone working the same account from the panel, where the allowance is 50 requests a day on Free and 500 on Pro, and they are what a looping assistant multiplies. Reports are cached for 15 minutes, so a re-read inside that window is free.

Every call from row 1 down also takes `customerId`, which is required and omitted from the table for width.

| # | Call | Arguments | Requests |
|---|---|---|---|
| 0 | `crm_list_google_ads_accounts` | none | 0 |
| 1 | `crm_google_ads_summary` | `range: "LAST_7_DAYS"` | 1 |
| 2 | `crm_google_ads_summary` | `range: "LAST_14_DAYS"` | 1 |
| 3 | `crm_google_ads_campaigns` | `range: "LAST_7_DAYS"` | 1 |
| 4 | `crm_google_ads_breakdown` | `level: "ad_groups", range: "LAST_7_DAYS"` | 1 |
| 5 | `crm_google_ads_breakdown` | `level: "keywords", range: "LAST_7_DAYS"` | 1 |
| 6 | `crm_google_ads_breakdown` | `level: "search_terms", range: "LAST_7_DAYS"` | 1 |
| | **Total for one account** | | **6** |

Optional extras, priced the same way: `crm_google_ads_campaigns` at `LAST_14_DAYS` for per-campaign week over week (1), a breakdown narrowed with `campaignId` (1 each, it is a different cache key), and `crm_google_ads_campaign_settings` per campaign (1 each, and this one is never cached). A second ad account costs a second set.

Six requests is a cheap report. It is cheap enough to run several times a day without thinking about it, and cheap enough that even the Free-plan panel cap of fifty would carry it eight times. What stops any of that being true is somebody looping the assistant. One thing does reset the cache: any write against that account, including a budget change made in the panel, invalidates every cached report for it, so the next read pays full price.

**Verify:** your client's tool call log shows exactly six calls that reached Google, one per row above. `dailyQuota.used` will not help you here, because on Business it comes back as `0` whatever you have spent. If the log shows more than six, something is calling in a loop.

## Getting the two windows right

There is no "previous 7 days" range. You derive it: pull `LAST_7_DAYS` and `LAST_14_DAYS`, then subtract. Both ranges end on the same day, which is the property that makes the subtraction valid, and the five ranges the tools accept are `LAST_7_DAYS`, `LAST_14_DAYS`, `LAST_30_DAYS`, `THIS_MONTH` and `LAST_MONTH`.

Subtract only additive counters. Subtracting one CTR from another produces a number with no meaning, and an assistant will do it happily unless you forbid it, so put that sentence in the prompt.

| Metric | Previous week from subtraction? | How |
|---|---|---|
| Cost | Yes | `cost(14) - cost(7)` |
| Impressions | Yes | `impressions(14) - impressions(7)` |
| Clicks | Yes | `clicks(14) - clicks(7)` |
| Conversions | Yes | `conversions(14) - conversions(7)` |
| Conversion value | Yes | `conversionsValue(14) - conversionsValue(7)` |
| CTR | No, recompute | `clicks / impressions` from the derived totals |
| Average CPC | No, recompute | `cost / clicks` from the derived totals |
| ROAS | No, recompute | `conversionsValue / cost` from the derived totals |

**Verify:** your derived previous week clicks, multiplied by the derived average CPC, come back to the derived cost within rounding. If they do not, a ratio got subtracted somewhere.

## Read the numbers exactly as they come back

`crm_google_ads_summary` is the only call that returns ROAS.

```json
{
  "cost": 8420.50, "impressions": 96412, "clicks": 3110,
  "ctr": 0.0323, "averageCpc": 2.71,
  "conversions": 41.0, "conversionsValue": 61300.00,
  "roas": 7.28, "currencyCode": "TRY", "range": "LAST_7_DAYS"
}
```

Three details in that payload decide whether your table is right.

**`ctr` is a ratio, not a percentage.** `0.0323` is 3.23 percent. Multiply once, in one place, and label the column.

**`conversions` is a floating point number and can be fractional.** Google counts modelled and multiple conversions per click, so `7.4` is a normal value, not a bug. Do not round it into a claim about seven people.

**`roas` is `conversionsValue / cost` and exists only at account level.** For a campaign or an ad group you compute it yourself from the row, and you say so in the source column.

`crm_google_ads_campaigns` returns rows ordered by cost, most expensive first, with removed campaigns excluded:

```json
{
  "count": 4, "range": "LAST_7_DAYS",
  "campaigns": [
    { "id": "22190044113", "name": "TR | Search | CRM", "status": "ENABLED",
      "cost": 4905.30, "impressions": 48900, "clicks": 1180,
      "conversions": 17.0, "conversionsValue": 28900.00,
      "dailyBudget": null, "budgetCurrency": null }
  ]
}
```

Note what is not in a campaign row: no CTR, no average CPC, and `dailyBudget` is null for Google accounts. Compute CTR and CPC from the counters, and read the real budget with `crm_google_ads_campaign_settings` when you need it, at one request per campaign.

`crm_google_ads_breakdown` carries one row shape for all four levels, so fields that do not apply to a level stay empty:

```json
{
  "count": 12, "level": "search_terms", "range": "LAST_7_DAYS",
  "rows": [
    { "level": "search_terms", "id": "crm software free download", "name": "crm software free download",
      "status": "NONE", "matchType": null, "campaignId": "22190044113", "campaignName": "TR | Search | CRM",
      "adGroupId": "178220043", "adGroupName": "CRM for agencies", "cost": 318.40, "impressions": 2140,
      "clicks": 44, "ctr": 0.0206, "averageCpc": 7.24, "conversions": 0.0, "conversionsValue": 0.00 }
  ]
}
```

A search term has no id of its own, so `id` repeats the term text, and `status` says whether that term is already a keyword or already excluded. At `level: "keywords"` the `matchType` field is populated and `id` is the criterion id. At `level: "ads"` a responsive search ad has no name, so the first headline is used as the label. Every breakdown is capped at 500 rows, ordered by cost descending, which matters for search terms more than anywhere else: on a busy account you are reading the 500 most expensive terms, not all of them. Say that in the report rather than implying the tail was reviewed.

**Verify:** the campaign rows sum to the account summary for cost, impressions, clicks and conversions. If they do not, one of the two calls used a different range.

## Best and worst, without fooling yourself

Ranking is where a weekly report earns or loses trust. Four rules cover it.

**Rank ad groups by cost per conversion, not by conversions.** An ad group with 12 conversions at 300 each is worse than one with 4 at 60. Compute it as `cost / conversions` and show both inputs beside it.

**Set a floor before you call anything a winner or a loser.** A useful default is 30 clicks or one week of spend above your target cost per conversion, whichever comes first. Below that floor the label is "not enough data yet", which is a real finding and stops the report inventing a story out of three clicks.

**Name the worst ad group by wasted spend.** Cost above your target cost per conversion with zero conversions is the cleanest definition, and it is the one that produces an action.

**For search terms, split the list in two.** Terms with conversions and no matching keyword are candidates to add. Terms with meaningful cost and zero conversions are candidates to exclude. Everything else waits for more data. Both lists go to a human: this session cannot write to the account, which is the point, and the write path is in [pause campaigns and move budget, safely](./03-pause-and-rebudget-safely.md).

**Verify:** every entry in your best and worst lists clears the floor you wrote down, and each one shows cost, clicks and conversions next to the label.

## What these tools do not return

Say this before your first report, because it decides several rows in the table.

Impression share and lost impression share, quality score, auction insights, and any segment by device, hour of day, geography or demographic are not returned by any tool here. Neither is search volume, keyword difficulty or a planner forecast: there is no keyword planner in this tool set. Per-ad rows exist at `level: "ads"`, but asset level reporting, which headline or description pulled the result, does not.

All of those live in the Google Ads UI. Export them for the same window and paste them in as a clearly labelled second source, or leave the row reading "not available". Both are honest. Filling the gap with an estimate is not, and an assistant asked for a complete report will produce one if you do not forbid it.

## When conversion tracking is thin

This is the section that keeps a weekly report from becoming fiction. Check three things before you write a sentence containing the word ROAS.

**Conversions are zero while clicks are in the thousands.** That is a tracking finding, not a performance finding. Write "conversion tracking recorded nothing this week, so no cost per conversion or ROAS can be stated", and make fixing tracking the first action. Reporting a cost per conversion of infinity, or quietly omitting the row, both mislead.

**Conversions exist but `conversionsValue` is zero.** Value tracking is not configured, so `roas` comes back as `0`. Zero ROAS does not mean no return, it means no value was recorded. Report cost per conversion instead, and mark ROAS "not measured".

**Conversions are small.** Under roughly 15 conversions a week, week over week movement is mostly noise. State the count next to every rate you quote, and prefer a 30 day range for anything you plan to act on.

Two more habits. Attribution windows mean last week's conversions can keep rising for days after you pull the numbers, so record the timestamp of the pull in the report and expect small upward revisions. And if your CRM records a different number of leads than Google records conversions, both numbers belong in the report with their sources named, because the gap between them is usually the most useful thing in it.

## The prompt

Paste this and edit the dates. The constraints are the whole point.

```text
Produce the weekly Google Ads report for customerId <id>.
Use range LAST_7_DAYS for this week and derive last week from LAST_14_DAYS.

STEP 1. Numbers only, no commentary. One markdown table:
metric | this week | last week | change | source.
The source column names the exact tool call, its arguments, and the window.
For a derived value write "derived: <the arithmetic>".
Derive last week ONLY by subtracting additive counters: cost, impressions,
clicks, conversions, conversionsValue. Never subtract a ratio. Recompute CTR,
average CPC and ROAS from the derived totals and say so.
CTR is returned as a ratio: state it as a percentage and label it.
For anything not in this dataset, write "not available" and name where it
lives. Impression share, quality score, device and hour segments, and search
volume are not available. Do not estimate. Do not use an industry benchmark.

STEP 2. Stop. Wait for me to confirm the table.

STEP 3. Then: campaign table, cost descending, with CTR and cost per
conversion computed per row. Then the three best and three worst ad groups by
cost per conversion, each showing cost, clicks and conversions, and each
clearing a floor of 30 clicks. Then up to five search terms to add and up to
five to exclude, with cost, clicks and conversions for each.

STEP 4. Narrative, maximum 350 words. Every claim points at a row above.
Mark every "why" as a hypothesis unless a number supports it. Name the one
thing that got worse. If nothing got worse, say the data shows no regression.

STEP 5. Up to three actions. Each names the metric it should move and the
number it has to beat.

If conversions are 0 for the account, stop after step 1 and report that
conversion tracking recorded nothing, rather than reporting performance.
```

## A worked example report

Every figure below is fictional and exists to show the shape and the internal consistency to demand. Do not benchmark against it.

**Account:** CRM Solid TR (5093114872), currency TRY. **Window:** `LAST_7_DAYS`. **Compared against:** the previous 7 days, derived from `LAST_14_DAYS`. **Pulled:** 2026-09-15T08:10Z.

| Metric | This week | Last week | Change | Source |
|---|---|---|---|---|
| Cost | 8,420.50 | 7,760.40 | +8.5% | `crm_google_ads_summary` LAST_7_DAYS; last week derived: 16,180.90 (LAST_14_DAYS) minus 8,420.50 |
| Impressions | 96,412 | 92,492 | +4.2% | same pair, derived: 188,904 minus 96,412 |
| Clicks | 3,110 | 2,894 | +7.5% | same pair, derived: 6,004 minus 3,110 |
| CTR | 3.23% | 3.13% | +0.10pp | recomputed per window: clicks / impressions |
| Average CPC | 2.71 | 2.68 | +1.1% | recomputed per window: cost / clicks |
| Conversions | 41 | 36 | +13.9% | same pair, derived: 77 minus 41 |
| Conversion value | 61,300 | 48,600 | +26.1% | same pair, derived: 109,900 minus 61,300 |
| Cost per conversion | 205.38 | 215.57 | -4.7% | recomputed per window: cost / conversions |
| ROAS | 7.28 | 6.26 | +1.02 | recomputed per window: conversionsValue / cost |
| Impression share | not available | not available | not available | not returned by any tool here. Google Ads UI, not exported |

**Campaigns, this week, cost descending.** Source: `crm_google_ads_campaigns` LAST_7_DAYS. CTR and cost per conversion computed per row.

| Campaign | Cost | Clicks | CTR | Conv | Cost/conv | Value | ROAS |
|---|---|---|---|---|---|---|---|
| TR \| Search \| CRM | 4,905.30 | 1,180 | 2.41% | 17 | 288.55 | 28,900 | 5.89 |
| TR \| Search \| Competitor | 1,760.40 | 402 | 2.00% | 4 | 440.10 | 6,100 | 3.47 |
| TR \| Search \| Brand | 1,180.20 | 1,402 | 6.33% | 19 | 62.12 | 24,800 | 21.01 |
| TR \| Search \| Integrations | 574.60 | 126 | 2.40% | 1 | 574.60 | 1,500 | 2.61 |

The four rows sum to the summary above on cost, clicks and conversions, which is the check that the two calls used the same window.

**Ad groups.** Source: `crm_google_ads_breakdown` level ad_groups, LAST_7_DAYS. Floor: 30 clicks.

Best: Brand exact, 742.10 cost, 905 clicks, 13 conversions, 57.08 per conversion. CRM for agencies, 1,980.40 cost, 470 clicks, 9 conversions, 220.04 per conversion.

Worst: Omnichannel inbox, 1,412.80 cost, 331 clicks, 0 conversions. Competitor names, 1,760.40 cost, 402 clicks, 4 conversions, 440.10 per conversion against a 250 target.

Below the floor, not ranked: Integrations long tail, 18 clicks.

**Search terms.** Source: `crm_google_ads_breakdown` level search_terms, LAST_7_DAYS, 12 rows returned, well under the 500 row cap.

Add: "whatsapp support panel", 18 clicks, 122.70 cost, 2 conversions, currently matching broadly through "customer support crm".

Exclude: "crm software free download", 44 clicks, 318.40 cost, 0 conversions. "excel customer tracking template", 31 clicks, 190.10 cost, 0 conversions. Together 508.50, which is 6.0 percent of the week's spend.

**Narrative.**

Spend rose 8.5 percent and conversion value rose 26.1 percent, so the week was more efficient rather than simply larger: cost per conversion fell from 215.57 to 205.38 and ROAS moved from 6.26 to 7.28. On 41 conversions this is a real move, though not a large enough sample to call a trend from two windows.

Brand carried the efficiency, as it usually does: 62.12 per conversion against an account average of 205.38. That is a reason to be careful reading the account average, not a reason to spend more on brand, since brand demand is created elsewhere.

What got worse: the Competitor campaign. It spent 1,760.40, 20.9 percent of the week, for 4 conversions at 440.10 each against a 250 target, and its single ad group is the worst on the account by that measure. The Omnichannel inbox ad group is the other candidate, with 1,412.80 spent and nothing recorded. Separately, two search terms of clear free-tool intent took 508.50 with no conversions, and excluding them is the cheapest change available this week.

Impression share, quality score and device splits are absent from this report because no tool here returns them. Read those rows as missing, not as flat.

**Actions.**

1. Exclude "crm software free download" and "excel customer tracking template" as campaign negatives. Target: hold conversions at 41 or better while cost falls by roughly 500.
2. Pause the Omnichannel inbox ad group unless somebody can name a reason to keep it. It has spent 1,412.80 across two weeks for nothing recorded.
3. Decide on Competitor. Either cut its budget so it costs less than 1,000 a week, or accept 440 per conversion in writing. Target for next week: cost per conversion under 300.

## Running it every week

Run it on the same weekday at roughly the same hour. A variable run date makes windows incomparable and hides weekly seasonality, and since `LAST_7_DAYS` is a rolling window, a report pulled on Tuesday and the next on Friday overlap by four days.

Save each week's tables as a plain markdown file in a repository, not in a chat history. That archive is the only baseline some of these numbers will ever have, because the tools offer no range longer than 30 days or last month, and anything you import from the Google Ads UI is gone from its default view soon enough. After six weeks, paste the files in together and ask which metrics move together and which one moves first. That is the point where reporting starts changing decisions rather than describing them.

One habit worth keeping: never run the report from the session that makes changes. Read read-only, decide with a person, then switch sessions for the changes themselves.

## Related reading

- [Connect Google Ads to your AI assistant](./01-connect-google-ads-to-your-assistant.md)
- [Find wasted Google Ads spend](./02-find-wasted-google-ads-spend.md)
- [Pause campaigns and move budget, safely](./03-pause-and-rebudget-safely.md)
- [Publish a campaign with a human approval step](./04-publish-a-campaign-with-approval.md)
- [Scopes, quotas and safety](./06-scopes-quotas-and-safety.md)
- [Tool reference](../reference/tools.md)
- [Repository index](../README.md)

External: the [MCP specification](https://modelcontextprotocol.io), the [CRM Solid MCP docs](https://docs.crmsolid.com/integrations/mcp/), the [npm package](https://www.npmjs.com/package/@crmsolid/mcp-server), and Google's reference for [date ranges in reporting queries](https://developers.google.com/google-ads/api/docs/query/date-ranges).
