# Publish a Google Ads Campaign From Your Assistant, With a Human Approval Step

Publishing a Google Ads campaign over MCP is the one workflow in this guide where the assistant is deliberately not in charge. The campaign is built by a person in the Pinlyx panel, validated against Google with `validateOnly` so nothing is created, approved by a person, and only then published by the assistant. What comes out the other end is a real campaign in the ad account, created paused, which somebody has to enable on purpose. This tutorial walks that whole path, shows the JSON-RPC calls behind it, and is explicit about the parts that are not automated and why.

Start with [connect Google Ads to your assistant](./01-connect-google-ads-to-your-assistant.md) if the server is not wired into your client yet. Two conventions before the examples. MCP tool output is camelCase. And a `tools/call` response wraps the payload as a JSON string in `result.content[0].text`, so every JSON block below is that inner payload after one more parse.

## The path, and who owns each step

| # | Step | Where it happens | Who acts | Reaches Google |
|---|---|---|---|---|
| 1 | Build the campaign | Ads Studio in the panel | A person | No |
| 2 | Local checks | Ads Studio, the **Check** button | Panel validator | No |
| 3 | Dry run | `crm_dry_run_ad_draft`, or **Test against network** | Assistant or person | Yes, 1 request |
| 4 | Fix what Google returned | Ads Studio | A person | No |
| 5 | Submit and approve | Ads Studio, **Submit for review** then **Approve** | Two people, ideally | No |
| 6 | Publish | `crm_publish_ad_draft` | Assistant | Yes, 1 request |
| 7 | Confirm | `crm_google_ads_campaign_settings` | Assistant | Yes, 1 request |
| 8 | Enable | `crm_set_google_ads_status` | A person decides, the assistant executes | Yes, 1 request |

## What is deliberately not automated

Two properties make the shape above defensible: the publish tool refuses any draft that is not in status `Approved`, so the approval is not advisory, and the campaign is created paused, so a mistaken publish costs you a cleanup rather than a budget. Each of the four gaps below is a hole somebody will otherwise assume is filled.

**No tool creates or edits a draft over MCP.** `crm_list_ad_drafts` is read-only, and the two write tools take exactly one argument each, a `draftId`. There is no `create_ad_draft`. An assistant cannot invent a campaign and push it at your account, because it has no way to author one. Campaigns are written in Ads Studio, at [app.crmsolid.com/ads-studio](https://app.crmsolid.com/ads-studio), under Insights > Ads > Ads Studio.

**Approval is not an MCP tool either.** Submit and approve exist only in the panel. That is the point: the reviewer is a person looking at a rendered campaign, not a model deciding it looks fine. Approval is also fragile on purpose. Editing an approved draft sends it back to `Draft` and the approval has to be redone, so nobody can sign off on one campaign and publish a different one.

**No image, video or asset upload.** Nothing in this tool set uploads a file. Image extensions, display creative and video assets are not reachable here. A campaign that needs them is finished in the Google Ads UI after publishing.

**No keyword planner.** Nothing returns search volume, competition or a forecast. Keywords come from your own account data, which is what [find wasted Google Ads spend](./02-find-wasted-google-ads-spend.md) is for, or from Google's own planner in the Ads UI.

## Before you start

You need a Pinlyx account with a connected Google Ads account, and an API key from [settings/developers](https://app.crmsolid.com/settings/developers) carrying both `ads:read` and `ads:write`. Keys are shown once, so store it before you close the dialog.

Connecting and using Google Ads inside Pinlyx is available on every plan, including Free, but the API key and the MCP server are Business-plan developer surfaces, so the MCP path needs Business. Ads Studio, where the draft in this tutorial is built and approved, is on the Business plan as well.

```jsonc
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

Note what is missing from that block. `--read-only` is not there, and it cannot be for this workflow: the local proxy drops every write tool, and both `crm_dry_run_ad_draft` and `crm_publish_ad_draft` are classed as writes. The dry run creates nothing, but it spends a request against your Google account and needs `ads:write`, so it is filtered out with the rest. Use a read-only session for the reporting work in [tutorial 05](./05-weekly-google-ads-report.md) and a separate session, with a separate key, for this one.

```bash
export CRMSOLID_API_KEY="csk_live_..."
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer $CRMSOLID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"crm_list_google_ads_accounts","arguments":{}}}'
```

**Verify:** the response names at least one account with a `customerId` of digits, and a `dailyQuota` object. That call reads the CRM database and never reaches Google, so it costs you nothing.

## Step 1: Build the campaign in Ads Studio

Open Ads Studio, choose **New campaign**, pick Google Ads as the network, and pick the ad account you will publish into. The builder is where every field lives, and the shape it produces is what the publish step turns into Google resources, in one atomic request:

- a campaign budget, with the daily amount and currency
- the campaign itself, with type, bidding strategy and schedule
- location and language criteria at campaign level, both include and exclude
- campaign negative keywords, which apply across every ad group
- ad groups, each with its own max CPC
- keywords with match types, per ad group
- responsive search ads: 3 to 15 headlines of up to 30 characters, 2 to 4 descriptions of up to 90 characters

Two fields decide how the rest of this tutorial goes. **Ad account** must be set, because a dry run with no account selected never leaves the building. And **Launch as** should stay on `Paused (recommended)`, which is the default.

Press **Check** when the form is full. That runs the panel's own validator locally: character limits, headline counts, duplicate headlines, missing landing pages. It is free and instant, and it catches most of what Google would reject anyway.

**Verify:** the checks panel reads "No problems found. This campaign is ready", or lists errors with the exact field each one belongs to. Errors block publishing. Warnings do not.

## Step 2: Find the draft from the assistant

`crm_list_ad_drafts` is how the assistant gets a `draftId`. Both filters are optional: `network` accepts `Google`, `Meta` or `TikTok`, and `status` accepts one of the lifecycle states, for example `Approved`.

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer $CRMSOLID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"crm_list_ad_drafts","arguments":{"network":"Google"}}}' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d).on('end',()=>console.log(JSON.parse(s).result.content[0].text))"
```

That `node` filter unwraps the MCP envelope and prints the tool payload, which looks like this:

```json
{
  "count": 2,
  "drafts": [
    { "id": 41, "network": "Google", "accountId": "5093114872", "name": "TR | Search | CRM",
      "objective": "leads", "status": "Draft", "dailyBudget": 400.00, "currency": "TRY",
      "updatedAt": "2026-09-14T11:20:44Z" }
  ]
}
```

The status moves Draft, In review, Approved, Published. The list does not carry the campaign spec, only the header fields, so an assistant reading it cannot see or reason about your headlines. That is a size decision rather than a security one, but it has the useful side effect that reviewing copy stays a panel job.

**Verify:** you have an integer `id` for the campaign you just built, and its `accountId` is not null. If `accountId` is null, go back to the builder and choose the ad account before continuing.

## Step 3: Ask Google to validate the whole thing

`crm_dry_run_ad_draft` sends the entire campaign to `customers/{id}/googleAds:mutate` with Google's own `validateOnly` flag set. Every resource goes in one request, each operation naming itself with a temporary resource id that later operations point at, so Google resolves budget to campaign to ad group to keyword server side and validates the structure as a whole. Nothing is created. This is the only way to get that answer: sending the layers as separate validate-only calls cannot work, because a validate-only budget returns no resource name for the campaign to reference.

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer $CRMSOLID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"crm_dry_run_ad_draft","arguments":{"draftId":41}}}'
```

Pipe it through the same `node` filter as above to read the payload.

A clean result:

```json
{ "validated": true, "issues": [], "error": null }
```

A result Google refused:

```json
{
  "validated": false,
  "issues": [
    { "field": "mutate_operations.adGroupAdOperation.create.ad.responsiveSearchAd.headlines", "severity": "error", "message": "Too few headlines were provided." },
    { "field": "campaign", "severity": "error", "message": "Google Ads API returned 400." }
  ],
  "error": "Google Ads API returned 400."
}
```

One behaviour to know before you trust a green result. If the draft has no ad account selected, the dry run does not contact Google at all. It runs the local validator and returns that, which can read as a pass while nothing was checked upstream. This is why step 2 tells you to confirm `accountId` first.

**Verify:** `validated` is `true` and `error` is `null`. Anything in `issues` with `severity: "error"` means Google would reject the publish. Entries with `severity: "warning"` come from the local validator and do not block anything.

## Step 4: Read the issues Google returns

There are two kinds of entry in `issues`, and telling them apart saves confusion. **Local issues** carry a dotted path into the campaign spec, in the shape `adGroups[0].ads[1].headlines[2]`. Those map straight onto the builder: ad group one, its second ad, the third headline. Fix them in the panel.

**Google's issues** carry Google's own field path, joined from the API's `fieldPathElements`, so they look like `mutate_operations.adGroupAdOperation.create.ad.finalUrls`. They name the operation type, which tells you the layer, and a `campaign` entry usually carries the top level summary rather than a specific field.

| What Google says | What it usually means | Where to fix it |
|---|---|---|
| Too few headlines were provided | Below the 3 headline minimum for a responsive search ad | Builder, Headlines |
| Line too long | A headline over 30 or a description over 90 characters | Builder, the flagged field |
| Duplicate headlines are not allowed | Two headlines identical after trimming and lowercasing | Builder, Headlines |
| The URL is invalid | Landing page missing, not absolute, or not reachable | Builder, Landing page |
| The budget amount is too low | Below the account currency's minimum daily budget | Builder, Budget |
| The customer account is not enabled | Billing is not set up on the ad account | Google Ads, not here |

The last row is the class of problem these tools cannot solve. Payment methods, monthly invoicing and account creation are not exposed here, and the Google Ads API does not offer the first two at all. Fix, then run the dry run again. Each attempt costs one request against Google, so read the whole issue list before editing rather than fixing one line at a time.

**Verify:** a repeat dry run returns `validated: true` with an empty `issues` array.

## Step 5: Submit and approve, in the panel

In Ads Studio, press **Submit for review**. The draft moves to In review, and the buttons change: a reviewer now sees **Approve** and **Reject**. Rejecting requires a note saying what has to change, which is the difference between a review process and a rubber stamp.

Have a different person approve it where you can. The one guarantee the system gives you is that the campaign that goes live is the campaign that was approved: any edit after approval resets the draft to `Draft` and clears the review, so an approved draft cannot quietly change underneath the reviewer.

What a reviewer should actually check, given the dry run already passed the mechanical parts: the daily budget and the currency, the landing pages, the geographic targeting, the campaign negative keywords, and whether `Launch as` is still Paused.

**Verify:** `crm_list_ad_drafts` with `{"status":"Approved"}` returns the draft, and its `id` is the one you dry ran.

## Step 6: Publish

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer $CRMSOLID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"crm_publish_ad_draft","arguments":{"draftId":41}}}'
```

```json
{
  "published": true,
  "campaignId": "22841009317",
  "note": "Created paused. Enable it with crm_set_google_ads_status when you are ready to spend.",
  "resources": [
    { "kind": "budget", "id": "customers/5093114872/campaignBudgets/14200311", "name": "TR | Search | CRM budget 20260915T081455" },
    { "kind": "campaign", "id": "customers/5093114872/campaigns/22841009317", "name": "TR | Search | CRM" },
    { "kind": "criterion", "id": "customers/5093114872/campaignCriteria/22841009317~2792", "name": null },
    { "kind": "ad_group", "id": "customers/5093114872/adGroups/178220043", "name": "CRM for agencies" },
    { "kind": "ad", "id": "customers/5093114872/adGroupAds/178220043~771002234", "name": "One inbox for every channel" }
  ]
}
```

The publish is atomic. Either every resource in that list exists afterwards or none does, because it is the same single mutate request as the dry run with `validateOnly` turned off.

Three refusals are worth recognising by their message. A draft that is not `Approved` is refused before anything is sent. A draft with no ad account is refused with a message telling you to choose one. And a rejection from Google at publish time leaves the draft in `Failed` with the platform's own reason recorded, which is rare after a clean dry run but happens when the account state changed in between, billing being the usual culprit.

**Verify:** `published` is `true` and `campaignId` is a numeric string. Keep the `resources` list. It is the only record of exactly what was created, and it is what you would use to clean up a campaign you did not mean to publish.

## Step 7: Confirm the campaign exists, and is paused

Do not trust the publish response alone. Read the account back:

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer $CRMSOLID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"crm_google_ads_campaign_settings","arguments":{"customerId":"5093114872","campaignId":"22841009317"}}}'
```

```json
{
  "id": "22841009317", "name": "TR | Search | CRM", "status": "PAUSED",
  "channelType": "SEARCH", "biddingStrategyType": "TARGET_SPEND",
  "startDate": "2026-09-16", "endDate": null,
  "targetGoogleSearch": true, "targetSearchNetwork": false, "targetContentNetwork": false,
  "dailyBudget": 400.00, "budgetCurrency": "TRY", "targetCpa": null, "targetRoas": null
}
```

This is the moment to check the two settings that quietly cost money: `targetSearchNetwork` and `targetContentNetwork`. A search campaign that arrives with the content network on will spend most of its budget on placements you did not intend, and it is far cheaper to notice that now than in next week's report. `crm_google_ads_campaign_settings` is the one read in this tool set that is never cached, so it always reflects the live account and always costs one request.

**Verify:** `status` is `PAUSED`, `dailyBudget` and `budgetCurrency` match what was approved, and the network flags are what you intended.

## Step 8: Enable it when you are ready

This is the step that starts spending money. It should be a decision somebody makes out loud, on a day when somebody is watching the account.

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer $CRMSOLID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":6,"method":"tools/call","params":{"name":"crm_set_google_ads_status","arguments":{"customerId":"5093114872","level":"campaigns","entityId":"22841009317","status":"enabled"}}}'
```

```json
{ "updated": true, "level": "campaigns", "entityId": "22841009317", "status": "enabled" }
```

`level` accepts `campaigns`, `ad_groups`, `ads` and `keywords`. For an ad or a keyword the `entityId` is `{adGroupId}~{id}`, which you build from the `adGroupId` and `id` fields of a `crm_google_ads_breakdown` row. `status` accepts `enabled`, `paused` and `removed`. Treat `removed` as permanent: it is permanent in the ad account, not just here. And note that enabling the campaign does not enable its ad groups if any of them were built paused, so check those with `crm_google_ads_breakdown` at `level: "ad_groups"` before concluding that nothing is serving.

**Verify:** call `crm_google_ads_campaign_settings` again and read `status: "ENABLED"`. Any write invalidates the cached reports for that account, so this read is fresh and costs one request.

## An operator prompt you can paste

Give the assistant the boundary explicitly. Models are agreeable, and a vague instruction to "get the campaign live" is exactly the sort of thing that turns into an unplanned enable.

```text
You can validate and publish Google Ads campaign drafts. You cannot create,
edit or approve one, and you must not enable anything.

  1. crm_list_ad_drafts. Report id, name, status, dailyBudget and currency
     before doing anything else.
  2. If accountId is null, stop and tell me. A dry run without an account
     never reaches Google and its result means nothing.
  3. crm_dry_run_ad_draft. Report every issue with its field, severity and
     message verbatim. Do not paraphrase Google's wording.
  4. Stop. I fix the draft and I approve it.
  5. Only when I say it is approved: crm_publish_ad_draft. Report campaignId
     and the full resources list.
  6. crm_google_ads_campaign_settings on that campaignId. Report status,
     dailyBudget, budgetCurrency, targetSearchNetwork, targetContentNetwork.

NEVER call crm_set_google_ads_status unless I ask for that exact change in
that message. Never say a campaign is live. State the status you read back.
```

## What the whole path costs

Only requests that reach Google count against the meter. MCP access requires Business, and Business has no daily Google Ads cap, so the table below prices a launch rather than measuring it against a ceiling. In the CRM panel the same meter allows 50 requests a day on Free and 500 on Pro.

| Step | Tool | Requests |
|---|---|---|
| Find the draft | `crm_list_ad_drafts` | 0, reads the CRM database |
| Dry run | `crm_dry_run_ad_draft` | 1 per attempt |
| Publish | `crm_publish_ad_draft` | 1 |
| Confirm settings, before and after enabling | `crm_google_ads_campaign_settings` | 1 each, never cached |
| Enable | `crm_set_google_ads_status` | 1 |

A clean launch costs five requests, plus one for every extra dry run. On a capped plan, over the limit the API answers HTTP 429 with a message saying the limit was reached and that it resets at midnight UTC.

## Failure modes worth designing against

**Approving, then editing.** The edit resets the draft to `Draft` and the publish is refused. This is correct behaviour and it still surprises people, so say it in your runbook.

**Enabling before the tracking works.** A campaign enabled before conversion tracking fires produces a week of spend with no conversion data, and the first report you write about it cannot say anything useful. Confirm conversions are recording on an existing campaign before you enable a new one.

**Assuming the assistant changed what it says it changed.** Every write tool returns the arguments it was given, not the state Google now holds. `{"updated": true}` means the mutate succeeded, and the read back in step 7 is what makes it true for you.

## Related reading

- [Connect Google Ads to your AI assistant](./01-connect-google-ads-to-your-assistant.md)
- [Find wasted Google Ads spend](./02-find-wasted-google-ads-spend.md)
- [Pause campaigns and move budget, safely](./03-pause-and-rebudget-safely.md)
- [A weekly Google Ads report your assistant can write](./05-weekly-google-ads-report.md)
- [Scopes, quotas and safety](./06-scopes-quotas-and-safety.md)
- [Tool reference](../reference/tools.md)
- [Repository index](../README.md)

External: the [MCP specification](https://modelcontextprotocol.io), the [Pinlyx MCP docs](https://docs.pinlyx.com/integrations/mcp/), the [npm package](https://www.npmjs.com/package/@crmsolid/mcp-server), and Google's own documentation for [mutating resources](https://developers.google.com/google-ads/api/docs/mutating/overview), which covers the grouped mutate and the temporary resource names the publish step relies on.
