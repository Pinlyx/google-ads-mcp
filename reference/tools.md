### `crm_google_ads_breakdown`

Performance one level below the campaign: ad groups, ads, keywords or search terms, each with impressions, clicks, CTR, average CPC, spend and conversions. This is where wasted spend shows up: read 'search_terms' to find queries worth excluding.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only, from crm_list_google_ads_accounts. |
| `level` | string | no | Which level to report on. One of: `ad_groups`, `ads`, `keywords`, `search_terms`. Default `ad_groups`. |
| `range` | string | no | Reporting window. One of: `LAST_7_DAYS`, `LAST_14_DAYS`, `LAST_30_DAYS`, `THIS_MONTH`, `LAST_MONTH`. Default `LAST_30_DAYS`. |
| `campaignId` | string | no | Limit the rows to one campaign (optional). |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_campaign_settings`

Live settings of one campaign: status, channel type, bidding strategy and its target, daily budget, networks and schedule. Read this before changing a budget or a bidding strategy.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only, from crm_list_google_ads_accounts. |
| `campaignId` | string | yes | Campaign id from crm_google_ads_campaigns. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_campaigns`

Campaigns in a Google Ads account with status, spend, impressions, clicks and conversions, most expensive first. Use to find the campaignId to drill into, pause or re-budget.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only, from crm_list_google_ads_accounts. |
| `range` | string | no | Reporting window. One of: `LAST_7_DAYS`, `LAST_14_DAYS`, `LAST_30_DAYS`, `THIS_MONTH`, `LAST_MONTH`. Default `LAST_30_DAYS`. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_summary`

Account-level Google Ads performance for a date range: spend, impressions, clicks, CTR, average CPC, conversions, conversion value and ROAS. Use for 'how did Google Ads do last month?'.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only, from crm_list_google_ads_accounts. |
| `range` | string | no | Reporting window. One of: `LAST_7_DAYS`, `LAST_14_DAYS`, `LAST_30_DAYS`, `THIS_MONTH`, `LAST_MONTH`. Default `LAST_30_DAYS`. |

Kind: read-only. Scope: `ads:read`.

### `crm_list_ad_drafts`

Campaign drafts built in Ads Studio, with network, status (Draft, In review, Approved, Published) and daily budget. Use to find the draftId to validate or publish.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `network` | string | no | Filter by ad network (optional). One of: `Google`, `Meta`, `TikTok`. |
| `status` | string | no | Filter by status, e.g. 'Approved' (optional). |

Kind: read-only. Scope: `ads:read`.

### `crm_list_google_ads_accounts`

List the Google Ads accounts connected to this workspace, with currency, time zone and today's remaining request allowance. Call this first: every other Google Ads tool needs a customerId from here.

No arguments.

Kind: read-only. Scope: `ads:read`.

### `crm_dry_run_ad_draft`

Ask the ad network to validate a draft campaign without creating anything. Returns the network's own field-level problems. Always do this before publishing.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `draftId` | integer | yes | Draft id from crm_list_ad_drafts. |

Kind: write. Scope: `ads:write`.

### `crm_publish_ad_draft`

Publish an APPROVED draft to the ad network. Creates real campaign structure, paused, so nothing spends until someone enables it. A draft that has not been approved in Ads Studio is refused.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `draftId` | integer | yes | Draft id from crm_list_ad_drafts, in status Approved. |

Kind: write, destructive. Scope: `ads:write`.

### `crm_set_google_ads_status`

Pause, enable or remove a Google Ads campaign, ad group, ad or keyword. Pausing stops spend immediately; removing is permanent in the ad account.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only, from crm_list_google_ads_accounts. |
| `level` | string | yes | What the entityId refers to. One of: `campaigns`, `ad_groups`, `ads`, `keywords`. |
| `entityId` | string | yes | Campaign or ad group id; for an ad or keyword use '{adGroupId}~{id}' as returned by crm_google_ads_breakdown. |
| `status` | string | yes | New state. One of: `enabled`, `paused`, `removed`. |

Kind: write, destructive. Scope: `ads:write`.

### `crm_update_google_ads_bidding`

Switch a campaign's bidding strategy, optionally with a target. Changing this changes how the account bids, so read crm_google_ads_campaign_settings first and confirm with the person.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only, from crm_list_google_ads_accounts. |
| `campaignId` | string | yes | Campaign id from crm_google_ads_campaigns. |
| `strategy` | string | yes | Bidding strategy to move to. One of: `manual_cpc`, `maximize_clicks`, `maximize_conversions`, `maximize_conversion_value`. |
| `targetCpa` | number | no | Target cost per conversion, for maximize_conversions (optional). |
| `targetRoas` | number | no | Target return on ad spend as a ratio, e.g. 3 for 300%, for maximize_conversion_value (optional). |

Kind: write. Scope: `ads:write`.

### `crm_update_google_ads_budget`

Set the daily budget of a Google Ads campaign, in the account's currency. Takes effect immediately and changes what the account can spend, so confirm the number with the person first.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only, from crm_list_google_ads_accounts. |
| `campaignId` | string | yes | Campaign id from crm_google_ads_campaigns. |
| `dailyBudget` | number | yes | New daily budget in the account currency. |

Kind: write. Scope: `ads:write`.
