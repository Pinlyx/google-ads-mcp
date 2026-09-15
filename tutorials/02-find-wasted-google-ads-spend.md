# Find Wasted Google Ads Spend With an AI Assistant

Wasted Google Ads spend is money that went to search queries you would never have bid on. It hides in the search terms report, one level below the keywords you actually chose, and it is invisible in any campaign-level chart. This tutorial builds the loop that finds it: pull search terms for a window, sort by cost among the rows with no conversions, judge each term for relevance, decide what to exclude, and decide what to do with the terms that did convert.

The assistant does the reading, the sorting and the first pass of judgement. You make the decisions, and you apply the exclusions yourself, because adding a negative keyword is not exposed as a tool on this server. Everything below is read-only: nothing here changes an ad account.

## What you need

| Requirement | Detail |
|---|---|
| MCP client | Any client that speaks MCP over stdio |
| Server | `@crmsolid/mcp-server`, the local bridge to `https://api.crmsolid.com/mcp` |
| Scope on the key | `ads:read` only. This whole tutorial is reads |
| Plan | Business. The MCP server and API keys are on the Business plan. Google Ads in the CRM is on every plan, including Free |
| Pinlyx account | Holds the ad account connection and the key |
| Daily allowance | None, because Business has no Google Ads cap. In the panel the same meter allows 50 requests a day on Free and 500 on Pro |
| Google Ads account | Connected by OAuth in the panel, see below |
| Time | About 25 minutes the first time, about 10 minutes weekly after that |

Connect the ad account first, in the Pinlyx panel: Insights > Ads > Google Ads > "Connect Google Ads account". The MCP tools read the account you connect there, so a key with `ads:read` and no connected account gives you tools that work and return nothing useful. Then create the key at [app.crmsolid.com/settings/developers](https://app.crmsolid.com/settings/developers) and add one entry to your client config. If none of that is in place yet, [01: Connect Google Ads to Your AI Assistant](./01-connect-google-ads-to-your-assistant.md) walks through it.

```jsonc
{
  "mcpServers": {
    "crmsolid-ads": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server", "--tools", "ads", "--read-only"],
      "env": { "CRMSOLID_API_KEY": "csk_live_..." }
    }
  }
}
```

`--tools ads` narrows the surface to the Google Ads family. `--read-only` keeps the six read tools and drops the five write tools before your client ever sees a list. Both filters run inside the local bridge, so a filtered tool is not listed and not callable.

Two conventions before the JSON starts. MCP tool output is camelCase. And `ctr` is a ratio, not a percentage: `0.0281` means 2.81 percent, and an assistant that reports "CTR 0.03 percent" has read the field and forgotten the unit.

## Step 1: Find the account and check today's allowance

```text
Call crm_list_google_ads_accounts. Show me each account with its customerId,
name, currency and time zone, then today's remaining request allowance.
```

The tool takes no arguments. It answers:

```json
{
  "count": 1,
  "accounts": [
    {
      "customerId": "4783920156",
      "name": "Northwind Tools",
      "currency": "EUR",
      "timeZone": "Europe/Amsterdam",
      "status": "Connected",
      "connectedAt": "2026-06-02T11:20:41Z",
      "lastSyncedAt": "2026-09-15T07:55:03Z"
    }
  ],
  "dailyQuota": { "used": 0, "limit": null, "remaining": null, "unlimited": true }
}
```

Two fields decide the rest of the session. `customerId` is required by every other Google Ads tool, digits only. `dailyQuota` is the daily meter, and that is the shape it takes on Business, the plan the MCP server requires: `limit` and `remaining` come back as `null`, `unlimited` is `true`, and `used` is reported as `0` whatever you have actually spent, because the status call returns the unlimited answer without reading the counter. On a capped plan the same block carries real numbers: `limit` 50 on Free or 500 on Pro, `remaining` counting down, and `unlimited` false.

This call never reaches Google. It reads the workspace's own connection records and a counter, so it costs nothing against the allowance. Run it as often as you like.

The same call without a client, so you can check the wiring outside any assistant:

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer $CRMSOLID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"crm_list_google_ads_accounts","arguments":{}}}' \
  | jq -r '.result.content[0].text' | jq .
```

The `jq` twice is not a typo. The envelope carries the payload as a JSON string inside `content[0].text`, so it has to be parsed twice:

```json
{"jsonrpc":"2.0","id":1,"result":{"content":[{"type":"text","text":"{\"count\":1,\"accounts\":[...]}"}],"isError":false}}
```

**Verify:** you have a `customerId` of digits only, and `dailyQuota` reads `unlimited: true`, which is what Business returns. A JSON-RPC error with code `-32002` and the text `Tool 'crm_list_google_ads_accounts' requires scope 'ads:read'` means the key is missing the scope. Grant it, then restart the client so the bridge picks the key up again.

## Step 2: Pull the search terms in one call

One call covers the whole account. Ask for more than you think you need, because a second call is another request against Google and a narrower one rarely answers the follow-up question.

```text
Call crm_google_ads_breakdown with customerId 4783920156, level "search_terms"
and range "LAST_30_DAYS". Do not summarise yet. Tell me how many rows came back
and the total cost across them.
```

```json
{
  "name": "crm_google_ads_breakdown",
  "arguments": {
    "customerId": "4783920156",
    "level": "search_terms",
    "range": "LAST_30_DAYS"
  }
}
```

```json
{
  "count": 214,
  "level": "search_terms",
  "range": "LAST_30_DAYS",
  "rows": [
    {
      "level": "search_terms",
      "id": "free crm software",
      "name": "free crm software",
      "status": "NONE",
      "matchType": null,
      "campaignId": "20194857362",
      "campaignName": "Search: Generic tools",
      "adGroupId": "1571940382",
      "adGroupName": "CRM generic",
      "cost": 184.20,
      "impressions": 3411,
      "clicks": 96,
      "ctr": 0.0281,
      "averageCpc": 1.92,
      "conversions": 0,
      "conversionsValue": 0
    }
  ]
}
```

One of 214 rows is shown. Every row carries the same fields, so a term that converted differs only in having `conversions` and `conversionsValue` above zero. Four things about this shape matter before you build on it.

Rows arrive ordered by cost, most expensive first, capped at 500. If `count` is exactly 500 the report is truncated and you are missing the tail, which is the moment to split the pass by campaign.

`range` takes one of `LAST_7_DAYS`, `LAST_14_DAYS`, `LAST_30_DAYS`, `THIS_MONTH`, `LAST_MONTH`. There is no custom date range on this surface, and switching windows is a fresh request.

At this level `id` is the search term text itself and `matchType` is always `null`, because a search term is not an entity in the account. The keyword that matched it is, and that distinction is what makes step 5 necessary.

`status` is Google's own view of the term: `ADDED` means it already exists as a keyword, `EXCLUDED` means a negative already catches it, `NONE` means neither. Read it before proposing anything, or you will spend a review cycle on exclusions that already exist.

The raw call, for verification outside a client:

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer $CRMSOLID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"crm_google_ads_breakdown","arguments":{"customerId":"4783920156","level":"search_terms","range":"LAST_30_DAYS"}}}' \
  | jq -r '.result.content[0].text' | jq '{count, level, range}'
```

**Verify:** `count` is greater than zero and `level` echoes `search_terms`. A `count` of 0 with no error means the window holds no data for this account, not that the call failed. Check the same window in the Google Ads interface before you go looking for a bug.

## Step 3: Sort by cost among the rows with no conversions

There is no server-side filter for "zero conversions". The tool returns the rows and the sorting happens on the rows you already have, which is exactly why step 2 pulled the whole window in one call.

```text
From the rows you just received, build one table of every row where conversions
is 0, sorted by cost, highest first. Columns: term, campaignName, adGroupName,
cost, clicks, impressions, ctr as a percentage to 2 decimals, averageCpc,
status.

Under the table give me three lines: the total cost of these zero-conversion
rows, that total as a percentage of the cost of all rows, and the row count.

Do not judge relevance yet. Do not propose any exclusions yet.
```

Two guards in that prompt earn their place. Asking for the totals forces the assistant to work from the whole set rather than the handful of rows it found interesting. Asking for CTR as a percentage stops a ratio being read out as a percentage later.

**Verify:** the row count in the table plus the number of rows with at least one conversion equals `count` from step 2. If it does not, the assistant dropped rows. Say "include every row with conversions 0, do not truncate" and run it again.

## Step 4: Judge relevance, with a cost floor

Now the assistant reads the terms as language rather than as rows. Give it a floor so it does not spend your attention on cents.

```text
Take the zero-conversion table. Ignore any row costing less than 25 EUR in the
window: too little evidence to act on. Put each remaining term in exactly one
bucket.

- irrelevant: we do not sell this. Different product, different industry, a
  job search, a free tool, a competitor's product name, a how-to question.
- relevant but unqualified: a real buyer might search this, but it reads as
  research rather than intent to buy.
- relevant: we sell exactly this and it simply has not converted yet.
- unsure: you cannot tell from the words alone. This is a valid answer.

Judge the term only, do not guess at intent you cannot see in the words, and
give one short reason per row quoting the part you judged on.
Output: term | cost | bucket | reason, in cost order.
```

The cost floor matters more than the bucket names. A 30 day window on a small account produces a long tail of one-click terms, and a table of 180 rows is not a review, it is a formality. Twenty five in the account currency is a starting point. Raise it until the table fits on one screen.

**Verify:** read the `irrelevant` rows and check that you agree with the top five. Disagreeing with one or two is normal. Disagreeing with four means the assistant does not know what you sell, and the fix is two sentences of context at the top of the prompt, not a better model.

## Step 5: Decide what to exclude, and add the exclusion yourself

**Adding a negative keyword is not exposed as an MCP tool on this server.** There is no tool that creates one, and no argument on any existing tool that does it as a side effect. Any assistant that offers to add negatives for you is describing something that does not exist. The exclusion is added by a person, in one of two places: the Google Ads interface, under the campaign or ad group's negative keywords, or the Pinlyx campaign builder in Ads Studio. What the assistant can usefully produce is the list you paste there.

```text
For every row in the irrelevant bucket, produce an exclusion list:
term | suggested match type | campaign or ad group level | why

- Exact match when only that precise phrase is wrong. Phrase match when the
  wrong part will recur, for example "free" or "jobs", and name the phrase.
- Never suggest a phrase negative that could block a term in the relevant
  bucket. Check that bucket first and name the row you checked against.
- Campaign level when the phrase is wrong for the whole campaign, ad group
  level when it is wrong only there.

Give me the terms as plain text I can paste, one per line, then the table.
```

The second rule is the one that prevents the expensive mistake. A phrase negative for "free" also blocks "free trial crm software", and the assistant will not notice unless you make it look.

**Verify:** take the plain text list into Google Ads, add the negatives, and keep the table with the reasons. Next month the table tells you why a term is missing, which is the difference between a negative list and a pile of decisions nobody can reconstruct.

What about the terms that did convert? Two cases matter. A converting term with `status` of `NONE` is not a keyword yet: it is being matched loosely through something broader, so you are paying broad-match prices for a query you already know works. Adding it as an exact match keyword is the same kind of manual step as a negative, in the Google Ads interface or the campaign builder, not a tool call. A converting term with `status` of `ADDED` is already covered, so leave it alone.

Before you propose promoting anything, check what is already in the account:

```text
Call crm_google_ads_breakdown with customerId 4783920156, level "keywords",
range "LAST_30_DAYS". For each converting search term from the previous table,
tell me whether a keyword with that exact text already exists, and show the
keyword's matchType, status, cost and conversions.
```

**Verify:** every term you plan to add as a keyword is absent from the keywords list, and every term you plan to exclude is absent from the converting set. Those two checks catch the two ways this loop hurts an account.

## Keeping the pass cheap

MCP access requires Business, and Business has no daily Google Ads cap, so nothing in this pass is rationed. The meter is still worth knowing, for three reasons: it is what the `dailyQuota` block reports, the same ad account may be used from the panel on a lower plan, where the allowance is 50 requests a day on Free and 500 on Pro, and the behaviour below is what separates a four-request session from a forty-request one.

The meter is per workspace per day, and only requests that actually reach Google count. That fact is what makes 50 workable for a panel user, and it is what keeps an MCP session cheap.

**What is free, and what costs one.** `crm_list_google_ads_accounts` reads connection records and the meter, so it never reaches Google and never costs a request. Each of `crm_google_ads_summary`, `crm_google_ads_campaigns`, `crm_google_ads_breakdown` and `crm_google_ads_campaign_settings` costs one request when it has to go to Google, and a report that spans several pages of results still costs exactly one.

**The 15 minute cache.** Report results are cached for 15 minutes, keyed by account, level, campaign filter and range. Repeat the identical call inside that window and you get the same rows for free. Change any part of the key and it is a new call: `LAST_7_DAYS` and `LAST_30_DAYS` are two entries, and one campaign filter is a different entry from no filter.

**Prefer one breakdown call over many.** A single `search_terms` call with no `campaignId` returns up to 500 rows across every campaign for one request. The same coverage split across six campaigns costs six. Only batch by campaign when the unfiltered call comes back with `count` at 500, which means the tail was truncated, or when you genuinely want to review one campaign in isolation.

**Any write clears the cache for that account.** Changing a status, a budget or a bidding strategy invalidates every cached report for that account, so the next read is a fresh request. That matters more in the write loop than here, and it is covered in [03: Let an AI Pause Campaigns and Move Budget, Safely](./03-pause-and-rebudget-safely.md).

A full pass of this tutorial, counted in requests that reach Google:

| Call | Requests |
|---|---|
| `crm_list_google_ads_accounts` | 0 |
| `crm_google_ads_summary`, LAST_30_DAYS | 1 |
| `crm_google_ads_campaigns`, LAST_30_DAYS | 1 |
| `crm_google_ads_breakdown`, search_terms, LAST_30_DAYS | 1 |
| `crm_google_ads_breakdown`, keywords, LAST_30_DAYS | 1 |
| Total | 4 |

Four requests for the whole pass, and a re-run inside 15 minutes is zero. Four also sits well inside the Free-plan panel cap of fifty, for anyone who works the same account from there. What actually runs a session up is a loop: an assistant told to "check each campaign" calls the breakdown once per campaign, once per window, and spends forty requests before it has said anything. Give it the call you want made, with the arguments you want, and ask it to report before it calls anything again. That discipline is worth keeping on Business too, where nothing stops the loop on your behalf.

On a capped plan, when the allowance is spent, the API answers HTTP 429 and the message says the limit resets at midnight UTC. Through MCP the same limit comes back as a tool error rather than a transport error, with the text `Daily Google Ads limit reached (50 requests). It resets at midnight UTC, or upgrade your plan for more.` on a Free-plan workspace. Do not retry it in a loop. Nothing changes until midnight UTC.

## The saved prompt

Paste this into your client as a saved prompt or a slash command. It is the whole loop with the arguments fixed, which is what stops the call count drifting week to week.

```text
Run the wasted spend pass on Google Ads.

1. Call crm_list_google_ads_accounts. Report the customerId, the currency and
   the dailyQuota block. If unlimited is false and remaining is under 5, stop
   and say so instead of continuing.
2. Call crm_google_ads_breakdown once: level "search_terms",
   range "LAST_30_DAYS", no campaignId. Report the row count and total cost.
3. Table of every row with conversions 0, sorted by cost, highest first, with
   totals underneath. Columns: term, campaign, ad group, cost, clicks, CTR as
   a percentage, average CPC, status.
4. Ignore rows under 25 in account currency. Bucket the rest: irrelevant,
   relevant but unqualified, relevant, unsure.
5. For the irrelevant rows only, produce a paste-ready exclusion list with a
   suggested match type, checked against the relevant bucket first.
6. Separately, list the terms that converted and have status NONE.

Rules: make exactly the calls listed above and none per campaign; ctr is a
ratio, so multiply by 100 before writing a percentage; you cannot add negative
keywords and no tool on this server does, so produce the list, say where I add
it, and stop there. Call no write tool.
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Tool 'crm_google_ads_breakdown' requires scope 'ads:read'` | Key lacks the scope | Grant `ads:read` at [settings/developers](https://app.crmsolid.com/settings/developers), then restart the client |
| `Google Ads account is not connected for this user.` | The `customerId` is not connected in this workspace, or has extra characters | Re-read `customerId` from `crm_list_google_ads_accounts`, digits only |
| `count` is exactly 500 | The report was truncated at the row cap | Split the pass by `campaignId`, or shorten the range |
| CTR reported as a fraction of a percent | `ctr` is a ratio and was printed as a percentage | Put the conversion rule in the prompt, as in step 3 |
| Request count climbs fast in one session | The assistant called the breakdown once per campaign | Name the exact calls in the prompt, as in the saved prompt |
| `Daily Google Ads limit reached` mid-pass | Today's allowance is spent, which happens on Free or Pro and never on Business | Wait for midnight UTC or upgrade. Repeating identical calls inside 15 minutes is free, so re-read what you already pulled |
| The assistant offers to add the negatives for you | No such tool exists on this server | Reject it. The list is produced here, the exclusion is added in Google Ads or the campaign builder |
| Terms you excluded last month appear again | The exclusion was added at ad group level and the term matched in another ad group | Check `status` on the row: `NONE` means nothing excludes it anywhere |

## Exercise

Run the pass twice with one variable changed: `range` set to `LAST_30_DAYS`, then to `LAST_7_DAYS`, and compare which terms appear in both. A term that is expensive and unconverting in both windows is a decision. A term that appears only in the 30 day window is usually one bad week that has already stopped.

Then raise the cost floor in step 4 from 25 to 50 and count how many rows survive. If the answer is two, your account does not have a search terms problem this month, and the honest output of the pass is a short note saying so.

## Related reading

- [03: Let an AI Pause Campaigns and Move Budget, Safely](./03-pause-and-rebudget-safely.md), the write loop, with confirmation and read-back
- [06: Scopes, Quotas and Safety for a Google Ads MCP Server](./06-scopes-quotas-and-safety.md)
- [Tool reference](../reference/tools.md): every argument, default and scope on this surface
- [docs.pinlyx.com/integrations/mcp/](https://docs.pinlyx.com/integrations/mcp/) and [@crmsolid/mcp-server on npm](https://www.npmjs.com/package/@crmsolid/mcp-server)
- [The MCP specification](https://modelcontextprotocol.io)
