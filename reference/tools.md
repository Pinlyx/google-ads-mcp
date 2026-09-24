## Read tools

### `crm_google_ads_account_alerts`

Health check of one Google Ads account: everything that stops or throttles its ads, as alerts sorted by severity (critical, warning, info), each with a code, a plain explanation, the entity it concerns and the tool to use next. Checks account status, advertiser identity verification, billing, disapproved and policy-limited ads, disapproved keywords, campaigns that are not serving with Google's reasons, conversion tracking, and the optimization score. Start here when an account shows no impressions or no spend.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only, from `crm_list_google_ads_accounts`. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_ad_details`

Full content and review state of Google Ads ads: responsive search ad headlines and descriptions with their pinned position, final and mobile URLs, display path, tracking template, final URL suffix and custom parameters, approval and review status with policy topics and evidence, ad strength with Google's suggestions to raise it, primary status and reasons.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `adId` | string | no | Bare id, the composite `{adGroupId}~{adId}` from `crm_google_ads_breakdown` (`level: "ads"`), or a full resource name. Without it, or narrowed by `adGroupId`/`campaignId`, the newest ads come first. |
| `adGroupId` | string | no | Narrows the search to one ad group (optional). |
| `campaignId` | string | no | Narrows the search to one campaign (optional). |
| `includeAssetPerformance` | boolean | no | Adds every asset's performance label (BEST, GOOD, LOW, LEARNING, PENDING) and source. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_ad_preview`

Heuristic diagnosis of whether a search could show an ad from this account, built entirely from the account's own settings (campaign status, budget, schedule, location and language targeting, keywords, negative keywords and ad approval). Not a live auction lookup; Google's own Ad Preview and Diagnosis tool is the authoritative source.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | no | Limits the check to one Search campaign; without it, every enabled Search campaign is checked, up to 50. |
| `geoTargetId` | string | no | From `crm_google_ads_geo_search`, or a free-text place name, sets the location context. |
| `languageId` | string | no | An id, a BCP-47 code such as `tr`, or an English name, also from `crm_google_ads_geo_search`. |
| `device` | string | no | One of `mobile`, `desktop`, `tablet`. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_assets`

Assets (ad extensions) of a Google Ads account and everywhere each one is linked: sitelinks, callouts, structured snippets, call, price, promotion, image, lead form, business name and logo, with content, approval status, policy topics, and every link's status, source and primary status.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `level` | string | no | Which links to read: account, campaign, ad group. Default is all levels. |
| `campaignId` | string | no | Filters to one campaign's links (optional). |
| `adGroupId` | string | no | Filters to one ad group's links (optional). |
| `types` | array | no | Filters by asset field type (optional). |
| `includePerformance` | boolean | no | Adds each link's metrics for the window (cost, ctr, conversionRate). |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_auction_insights`

Auction insights for Search: which other advertisers (by display domain) competed in the same auctions and how the account compares: impression share, overlap rate, position-above rate, top and absolute-top rates, and the account's outranking share. Google only serves this to allowlisted API projects; otherwise the answer is `feature_not_available`.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | no | Narrows to one campaign; default is account level. |
| `adGroupId` | string | no | Narrows to one ad group (optional). |
| `range` | string | no | Reporting window over which insights are computed. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_audiences`

Audience (user) lists of a Google Ads account, newest first: id, name, type, description, membership status and life span, estimated size for Search and Display, eligibility, the Customer Match match rate, upload key type, closing reason, and the status of the latest Customer Match upload.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_bidding_strategies`

Portfolio (shared) bidding strategies of the account: type, status, targets and limits in the account currency, and which campaigns use each one, including strategies shared from a manager account.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_billing`

Billing state of one Google Ads account: the billing setup, whether it is on monthly invoicing, its account budgets (approved, proposed and adjusted limits, amount served, remaining, pending change), and the invoices issued this month and last month. Promotions are not available through the Google Ads API.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_breakdown`

Performance one level below the campaign, with the diagnostics needed to act on it: `ad_groups`, `ads`, `keywords` (including Quality Score), `search_terms`, `landing_pages`, `asset_groups`, `placements`, `videos`, `products`, `assets` or `paid_organic`. Filters run before paging; totals cover every filtered row. Defaults to `LAST_30_DAYS`, cost highest first, 50 rows per page.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `level` | string | yes | One of `ad_groups`, `ads`, `keywords`, `search_terms`, `landing_pages`, `expanded_landing_pages`, `asset_groups`, `placements`, `videos`, `products`, `assets`, `paid_organic`. |
| `range` | string | no | Reporting window. Default `LAST_30_DAYS` (ends yesterday). |
| `campaignId` | string | no | Limits the rows to one campaign (optional). |
| `limit` | integer | no | Rows per page; default 50. |
| `cursor` | string | no | From a previous response's `nextCursor`, to page further. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_budget_pacing`

Today's Google Ads spend against budget for every enabled campaign (or one campaignId): daily budget, whether it is shared, spend today/yesterday/this month, expected spend by now, pace, monthly cap, projected month spend, and whether the campaign is limited by budget.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | no | Narrows to one campaign; without it, every enabled campaign is covered. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_campaign_settings`

Live settings of one campaign: status with a plain-English explanation, channel type, the normalized bidding strategy with its target or CPC limit, the daily budget including whether it is shared, start and end dates, network settings, geo target type, ad schedule and language targets.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | yes | Campaign id from `crm_google_ads_campaigns`. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_campaigns`

Campaigns of a Google Ads account with their settings and performance for a window, including campaigns with no traffic: status, primary status and reasons, channel type, bidding strategy, daily budget, dates, optimization score, full metrics and search impression share. Sorted by cost, highest first.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `range` | string | no | Reporting window. Default `LAST_30_DAYS`. |
| `status` | string | no | Pass `all` to also list removed campaigns. |
| `sortBy` | string | no | Changes the default cost-descending order. |
| `limit` | integer | no | Rows per page. |
| `cursor` | string | no | From a previous response, to page further. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_change_history`

Who changed what in a Google Ads account, newest first. Merges Google's own change history (last 30 days) with changes made through Pinlyx tools, including whether `crm_google_ads_undo` can still revert each one.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `days` | integer | no | 1 to 30, default 7, counting today. |
| `startDate` | string | no | Alternative to `days`; must fall within the last 30 days. |
| `endDate` | string | no | Alternative to `days`; must fall within the last 30 days. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_conversion_actions`

Conversion tracking health of a Google Ads account: account-level settings and every non-removed conversion action with category, type, status, counting type, attribution model, windows, last conversion, conversions in the window, and a recording health verdict. Flags risky setups.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | no | Narrows the conversions-in-window figure to one campaign (optional). |
| `range` | string | no | Reporting window for conversions in the window. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_experiments`

Campaign experiments (A/B tests Google Ads runs by splitting a campaign's traffic): status in plain words, dates, promote status, the control and treatment arms, the traffic split, and the in-design campaign of an experiment still in setup.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `experimentId` | string | no | Reads one experiment in full, including background errors of a failed schedule or promotion. |
| `includeMetrics` | boolean | no | Compares treatment with control over the experiment's own dates, at most 5 experiments per call. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_forecast`

Keyword Planner forecast for a new Search campaign with the keywords given: expected clicks, cost and average CPC over the forecast period (default the next 30 days), plus conversions and cost per conversion with `maximize_conversions`, and per-day averages. Ignores the account's own history.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `keywords` | array | yes | The keywords to forecast. |
| `bidStrategy` | string | yes | One of `manual_cpc`, `maximize_clicks`, `maximize_conversions`. |
| `maxCpcBid` | number | no | Used with `manual_cpc` (optional `dailyBudget`) or to cap CPC under `maximize_clicks`. |
| `dailyBudget` | number | no | Required for `maximize_clicks` and `maximize_conversions`; optional for `manual_cpc`. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_geo_search`

Find Google Ads location ids by name, or look up known ids, for use with `crm_update_google_ads_targeting` and `crm_google_ads_ad_preview`. A purely numeric entry is read as a geoTargetId and returned with its current name.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `query` | string | no | One name to search for. |
| `names` | array | no | Several names, up to 25 total between `query` and `names`. |
| `countryCode` | string | no | Two letters, e.g. `TR`, narrows a name search to one country. |
| `targetTypes` | array | no | Filters by kind: Country, Region, City, Postal Code, Airport and other Google-defined kinds. |
| `locale` | string | no | Language of the returned names. Default `en`. |
| `includeLanguages` | boolean | no | Also returns the language id catalog. |
| `languageQuery` | string | no | Filters the language catalog by name, BCP-47 code, or exact id. |

Kind: read-only. Scope: `ads:read`. No `customerId`: this is a standalone lookup, not scoped to one account.

### `crm_google_ads_keyword_ideas`

Keyword Planner ideas: new keyword ideas from up to 20 seed keywords, a page URL, both, or a whole site, each with average monthly searches, competition, the top-of-page bid range, and the last 12 months of search volume, sorted by searches. Results are cached for 12 hours.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `keywords` | array | no | Up to 20 seed keywords. |
| `pageUrl` | string | no | A page URL, or a whole site, to derive ideas from instead of or alongside `keywords`. |
| `historicalOnly` | boolean | no | Returns the same metrics for exactly the keywords given (up to 1000), skipping new ideas. |
| `language` | string | no | For example `1037` or `tr`. Without it, Google uses all languages. |
| `geoTargetConstants` | array | no | For example `2792` or `TR`. Without it, Google uses all locations. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_labels`

List the account's labels: organizational tags with a name and an optional color and description.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `labelId` | string | no | Reads one label only (optional). |
| `includeAssignments` | boolean | no | Adds every campaign, ad group, ad and keyword that carries each label. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_negative_keywords`

List negative keywords of a Google Ads account: campaign-level, ad group-level, and shared negative keyword lists together with the campaigns each list is attached to.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | no | Campaign and ad group negatives of that campaign (optional). |
| `adGroupId` | string | no | Also resolves its campaign (optional). |
| `sharedSetId` | string | no | Reads that one shared list only (optional). |
| `includeListMembers` | boolean | no | Adds each shared list's own keywords. Default true only when `sharedSetId` is given. |
| `contains` | string | no | Case-insensitive substring filter on keyword text. |
| `limit` | integer | no | Rows per section. Default 200, max 5000. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_recommendations`

Google's recommendations for one account (the Recommendations page): type, campaign and ad group, the estimated weekly impact before and after applying, and the details Google gives per type. Also returns the account and campaign optimization scores.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | no | Filters to one campaign (optional). |
| `types` | array | no | Filters by recommendation type (optional). |
| `includeDismissed` | boolean | no | Shows dismissed recommendations too. Default false. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_segments`

Google Ads performance split by one dimension, for the account, one campaign or one ad group: device, network, hour of day, day of week, slot, country, region, city, location type, age, gender or conversion action. Rows carry the full metric set and their share of cost and clicks, highest cost first.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `dimension` | string | yes | One of `device`, `network`, `hour_of_day`, `day_of_week`, `slot`, `country`, `region`, `city`, `location_type`, `age`, `gender`, `conversion_action`. |
| `campaignId` | string | no | Narrows to one campaign (optional). |
| `adGroupId` | string | no | Narrows to one ad group (optional). |
| `range` | string | no | Reporting window. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_summary`

Account-level Google Ads performance for a window: spend, impressions, clicks, CTR, average CPC, interactions, conversions, conversion rate, cost per conversion, conversion value, ROAS, invalid clicks, search impression share, and the account optimization score.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. A manager (MCC) account has no metrics; use one of its client accounts. |
| `range` | string | no | Reporting window. Default `LAST_30_DAYS`. |
| `compareTo` | string | no | `previous_period` or `same_period_last_year`, adds the same metrics for that window and the fractional change. |
| `includeConversionsByAction` | boolean | no | Splits conversions by conversion action. |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_targeting`

Full targeting of one campaign: locations (included, excluded, bid modifiers, proximity/radius), the location option, languages, ad schedule, device bid modifiers, and audiences and demographics with their mode.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | yes | Campaign id from `crm_google_ads_campaigns`. |
| `includeAdGroups` | boolean | no | Also reads ad-group-level audiences and demographics (an extra query). |

Kind: read-only. Scope: `ads:read`.

### `crm_google_ads_timeseries`

Google Ads performance over time for the account, one campaign or one ad group: a point per day, week, month, or hour, or a profile by hour of day or day of week, in the account time zone. Every period of the window is present, even with zero traffic.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | no | Narrows to one campaign (optional). |
| `adGroupId` | string | no | Narrows to one ad group (optional). |
| `granularity` | string | yes | One of `day`, `week`, `month`, `hour`, `hour_of_day`, `day_of_week`. |
| `range` | string | no | Reporting window. Day series up to 400 days, hour series up to 14 days; `ALL_TIME` is not accepted. |
| `compareTo` | string | no | `previous_period` or `same_period_last_year`, aligns `comparison.points` by position. |
| `metrics` | array | no | Narrows each point to some fields, e.g. `["cost","clicks","conversions"]`. |

Kind: read-only. Scope: `ads:read`.

### `crm_list_google_ads_accounts`

List the Google Ads accounts connected to this workspace and whether each one can be used: account status, manager status and client accounts, connection status, currency, time zone, and today's request allowance. Call this first: every other Google Ads tool needs a `customerId` from here.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `refresh` | boolean | no | Account details are cached for about 6 hours; `true` reads them from Google again (one request per account). |

Kind: read-only. Scope: `ads:read`.

## Write tools

### `crm_google_ads_add_keywords`

Add keywords to a Search ad group.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `adGroupId` | string | no | Required unless every keyword comes from `fromSearchTerms` with its own ad group reference. |
| `keywords` | array | no | Keywords to add (text plus optional match type). |
| `matchType` | string | no | Default match type for items that do not choose their own. Falls back to `PHRASE` for `keywords`, `EXACT` for `fromSearchTerms`. |
| `cpcBid` | number | no | Default max CPC, account currency, not micros. Omit to use the ad group's bid. |
| `finalUrl` | string | no | Default landing page for items that do not give their own. |
| `status` | string | no | `ENABLED` (default) or `PAUSED` for every keyword added. |
| `fromSearchTerms` | array | no | References into a search term report, each with its own ad group. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it. |

Kind: write. Scope: `ads:write`.

### `crm_google_ads_add_negative_keywords`

Add negative keywords at the campaign, ad group or shared-list level.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `level` | string | no | `campaign`, `ad_group` or `shared_list`. Inferred from which id is given when omitted. |
| `campaignId` | string | no | Required for level `campaign` unless every item comes from `fromSearchTerms`. |
| `adGroupId` | string | no | Required for level `ad_group` unless every item comes from `fromSearchTerms`. |
| `sharedSetId` | string | no | Required for level `shared_list`. |
| `keywords` | array | no | Negative keywords to add (text plus optional match type). |
| `matchType` | string | no | Default match type. Falls back to `EXACT`. |
| `fromSearchTerms` | array | no | References into a search term report. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it. |

Kind: write. Scope: `ads:write`.

### `crm_google_ads_apply_label`

Create a label, or assign it to, or unassign it from, campaigns, ad groups, ads and keywords.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `action` | string | yes | `create`, `assign` or `unassign`. |
| `labelId` | string | no | From `crm_google_ads_labels`, to say which label for `assign`/`unassign`. |
| `labelName` | string | no | Names the label for `create`, or for `assign` (creates it if it does not exist yet). Must not repeat an existing label's name on `create`. |
| `color` | string | no | Hex value such as `#1A73E8`, for `create`. |
| `description` | string | no | At most 200 characters, for `create`. |
| `targets` | array | no | Campaigns, ad groups, ads or keywords, for `assign`/`unassign`. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it. |

Kind: write. Scope: `ads:write`.

### `crm_google_ads_apply_recommendation`

Apply or dismiss Google recommendations by id (from `crm_google_ads_recommendations`). There is no automatic undo.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `ids` | array | yes | Up to 100 recommendation ids per call; each succeeds or fails on its own. |
| `confirm` | boolean | yes for applying | Required to apply a recommendation for real. Not required to dismiss. |
| `validateOnly` | boolean | no | Previews what would be applied or dismissed, since Google has no validate-only mode for recommendations. |

Kind: write, destructive. Scope: `ads:write`.

### `crm_google_ads_bulk`

Apply many keyword-family changes in one call: status changes, bid changes, adding a keyword, adding or removing a negative keyword.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `operations` | array | yes | Up to 5000 items, each `{type, ...}` (`set_status`, `set_bid`, `add_keyword`, `add_negative`, `remove_negative`). |
| `partialFailure` | boolean | no | Default true: each operation succeeds, is skipped, or fails on its own. `false` is all-or-nothing, at most 1000 operations. |
| `confirm` | boolean | yes for removals | Required when any operation removes something (`status: REMOVED` or `remove_negative`). |
| `validateOnly` | boolean | no | Checks every operation with Google without applying anything. |

Kind: write, destructive. Scope: `ads:write`.

### `crm_google_ads_campaign_to_draft`

Copy a live Google Search campaign into a new Ads Studio draft (status Draft, campaign paused) that a person can edit, approve and publish. Creates nothing in Google Ads.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | yes | The live Search campaign to copy. |

Kind: write. Scope: `ads:write`.

### `crm_google_ads_create_ad_group`

Create a new ad group in a Search or Display campaign, optionally with its first keywords, all in one atomic call.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | yes | Campaign to create the ad group in. |
| `name` | string | yes | Must be unique within the campaign. |
| `status` | string | no | `PAUSED` (default) or `ENABLED`. |
| `cpcBid` | number | no | Default max CPC, account currency, not micros. Omit for no default bid. |
| `keywords` | array | no | Search campaigns only, same shape as `crm_google_ads_add_keywords`. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it. |

Kind: write. Scope: `ads:write`.

### `crm_google_ads_create_asset`

Create an asset (ad extension) and optionally link it in the same atomic call. `type` selects the shape: `sitelink`, `callout`, `structured_snippet`, `call`, `price`, `promotion` or `image` (fetched from a public https URL, at most 5 MB), each with its own required fields.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `type` | string | yes | `sitelink`, `callout`, `structured_snippet`, `call`, `price`, `promotion` or `image`, each with type-specific fields. |
| `name` | string | no | Optional label for any type. |
| `currencyCode` | string | no | For `price`/`promotion` amounts; defaults to the account currency. |
| `linkTo` | object | no | Links it at account, campaign or ad group level. Without it, the asset is created but does not serve. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it. |

Kind: write. Scope: `ads:write`.

### `crm_google_ads_create_bidding_strategy`

Create a portfolio (shared) bidding strategy that several campaigns can use, and optionally attach existing campaigns to it in the same atomic call.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `name` | string | yes | Must be unique in the account. |
| `type` | string | yes | `maximize_clicks`, `maximize_conversions`, `maximize_conversion_value`, `target_impression_share`, `target_cpa` or `target_roas`. |
| `targetCpa` | number | no | For `target_cpa` (required) or `maximize_conversions` (optional). |
| `targetRoas` | number | no | For `target_roas` (required) or `maximize_conversion_value` (optional). |
| `cpcBidCeiling` | number | no | Optional cap for most types. |
| `attachCampaignIds` | array | no | Moves those campaigns onto the new strategy immediately, replacing whatever bidding they had. |

Kind: write. Scope: `ads:write`.

### `crm_google_ads_create_rsa`

Create a responsive search ad in a Search ad group. Everything is validated first; nothing reaches Google until all checks pass. Created paused by default.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `adGroupId` | string | yes | Ad group to create the ad in. |
| `headlines` | array | yes | 3 to 15 headlines, at most 30 characters each. |
| `descriptions` | array | yes | 2 to 4 descriptions, at most 90 characters each. |
| `finalUrl` | string | yes | The ad's landing page, https. |
| `path1` | string | no | Display path segment, at most 15 characters. |
| `path2` | string | no | Display path segment, at most 15 characters; needs `path1`. |
| `replaceAdId` | string | no | Pauses that ad of the same ad group in the same atomic call. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it. |

Kind: write. Scope: `ads:write`.

### `crm_google_ads_customer_match`

Sync CRM contacts into a Google Ads Customer Match list, so campaigns can target, exclude or find people like existing customers. Emails and phones are normalised and SHA-256 hashed before sending.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `contactFilter` | object | yes | Picks workspace contacts (contactIds, tagIds, tagNames, stage, pipelineId, stageId, createdAfter). `{}` means every contact, limit up to 100000. |
| `listId` | string | no | An existing list from `crm_google_ads_audiences`. |
| `listName` | string | no | Creates a CRM-based list when none has that name. |
| `mode` | string | no | `add`, `remove` or `replace`. |
| `consent` | string | no | Only as the person states it; default `UNSPECIFIED`. |
| `confirm` | boolean | yes for real uploads | Required because it sends customer data to Google. |
| `validateOnly` | boolean | no | Dry run: counts contacts and identifiers, sends nothing. |

Kind: write, destructive. Scope: `ads:write`.

### `crm_google_ads_link_asset`

Link an existing asset to the account, a campaign or an ad group under a field type, or remove that link.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `assetId` | string | yes | From `crm_google_ads_assets` or `crm_google_ads_create_asset`. |
| `action` | string | yes | `link` or `unlink`. |
| `level` | string | yes | Account, campaign or ad group. |
| `campaignId` | string | no | Required when `level` is campaign or ad group. |
| `adGroupId` | string | no | Required when `level` is ad group. |
| `fieldType` | string | yes | The asset field type to link under. |
| `confirm` | boolean | yes for unlink | Required to remove a link for real. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it. |

Kind: write, destructive. Scope: `ads:write`.

### `crm_google_ads_manage_experiment`

Create and run a campaign experiment (A/B test) on a Search or Display campaign.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `action` | string | yes | `create`, `schedule`, `end`, `promote`, `graduate` or `remove`. |
| `baseCampaignId` | string | for `create` | Campaign whose traffic is split between control and treatment. |
| `trafficSplit` | number | no | Treatment's percent of traffic, default 50, for `create`. |
| `experimentId` | string | for non-create actions | The experiment to act on. |
| `scheduleNow` | boolean | no | Schedules right away on `create`. |
| `dailyBudget` | number | no | For `graduate`; defaults to the base campaign's. |
| `confirm` | boolean | yes for end/promote/graduate/remove | Those four have no undo. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it. |

Kind: write, destructive. Scope: `ads:write`.

### `crm_google_ads_manage_negative_list`

Create, attach, detach, rename or delete a shared negative keyword list.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `action` | string | yes | `create`, `attach`, `detach`, `rename` or `delete`. |
| `sharedSetId` | string | for all but `create` | The list to act on. |
| `name` | string | for `create`/`rename` | Must not repeat an existing list's name. |
| `campaignIds` | array | no | For `create` (initial attachment) or `attach`/`detach`. |
| `keywords` | array | no | Initial keywords for `create`, same shape as `crm_google_ads_add_negative_keywords`. |
| `matchType` | string | no | Default match type for `create` keywords. Falls back to `EXACT`. |
| `confirm` | boolean | yes for `delete` | Deleting removes the list, its keywords, and every campaign link; cannot be undone. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it; does not need `confirm`. |

Kind: write, destructive. Scope: `ads:write`.

### `crm_google_ads_remove_negative_keywords`

Remove negative keywords at the campaign, ad group or shared-list level. Removing one can let the searches it blocked start matching again.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `ids` | array | yes | `resourceId` or `resourceName` values from `crm_google_ads_negative_keywords`. |
| `level` | string | no | Narrows an ambiguous bare `resourceId`. |
| `confirm` | boolean | yes for real removal | Without it, the call answers with what would be removed and changes nothing. |
| `validateOnly` | boolean | no | Checks the removal with Google without applying it; does not need `confirm`. |

Kind: write, destructive. Scope: `ads:write`.

### `crm_google_ads_save_conversion_action`

Create a Google Ads conversion action, or change an existing one.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `conversionActionId` | string | no | Omit to create; pass to update. |
| `name` | string | for create | Required when creating. |
| `category` | string | for create | Required when creating. |
| `type` | string | no | Defaults to `UPLOAD_CLICKS`, the type `crm_google_ads_upload_conversions` needs. |
| `status` | string | no | `ENABLED` or `HIDDEN` (stops recording). |
| `primaryForGoal` | boolean | no | `false` makes it secondary so bidding ignores it. |
| `countingType` | string | no | `ONE_PER_CLICK` suits leads, `MANY_PER_CLICK` suits sales. |
| `defaultValue` | number | no | In `currencyCode`, default the account currency. |
| `currencyCode` | string | no | Defaults to the account currency. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it. |

Kind: write. Scope: `ads:write`.

### `crm_google_ads_undo`

Revert a Google Ads change made through Pinlyx tools, by the `changeId` that write returned.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `changeId` | string | yes | From the original write's result, or from `crm_google_ads_change_history`. |
| `validateOnly` | boolean | no | Checks the revert with Google without applying it. |

Kind: write. Scope: `ads:write`. Removals, validate-only calls, and changes older than 30 days cannot be undone.

### `crm_google_ads_update_ad`

Edit an existing ad in place; it keeps its id and history.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `adId` | string | yes | The ad to edit. |
| `finalUrls` | array | no | Replaces the current list. |
| `finalMobileUrls` | array | no | Replaces the current list; empty array clears it. |
| `path1` | string | no | Display path segment. |
| `path2` | string | no | Display path segment. |
| `trackingUrlTemplate` | string | no | Empty string clears it. |
| `finalUrlSuffix` | string | no | Empty string clears it. |
| `headlines` | array | no | Responsive search ads only; replaces the whole current list. |
| `descriptions` | array | no | Responsive search ads only; replaces the whole current list. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it. |

Kind: write. Scope: `ads:write`.

### `crm_google_ads_update_ad_group`

Rename an ad group and/or change its tracking template or final URL suffix.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `adGroupId` | string | yes | The ad group to edit. |
| `name` | string | no | Must stay unique within the campaign. |
| `trackingUrlTemplate` | string | no | Empty string clears it. |
| `finalUrlSuffix` | string | no | Empty string clears it. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it. |

Kind: write. Scope: `ads:write`.

### `crm_google_ads_update_keyword`

Change a keyword's match type and/or its own final URL.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `keywordId` | string | yes | Criterion id, or the composite `{adGroupId}~{criterionId}`. |
| `adGroupId` | string | no | Disambiguates a bare criterion id that exists in more than one ad group. |
| `matchType` | string | no | Changing it creates a new keyword id, returned as `newKeywordId`; labels are not copied over. |
| `finalUrl` | string | no | Empty string clears it back to the ad's URL. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it. |

Kind: write. Scope: `ads:write`.

### `crm_google_ads_upload_conversions`

Send offline conversions (for example won CRM deals) to an `UPLOAD_CLICKS` conversion action, so Smart Bidding learns from real results. Cannot be undone.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `conversionActionId` | string | yes | Must be `UPLOAD_CLICKS` type. |
| `conversions` | array | yes | At most 2000 per call. Each needs a click id (gclid/gbraid/wbraid) or an email/phone, plus `conversionTime`. |
| `contactId` | string | no | Fills email, phone and the ad click the CRM recorded for that contact. |
| `dealId` | string | no | Fills value, currency, won time, contact and a stable `orderId` so the same deal cannot be uploaded twice. |
| `validateOnly` | boolean | no | Try this first; uploads cannot be undone. |

Kind: write, destructive. Scope: `ads:write`.

### `crm_set_google_ads_status`

Pause, enable or remove one or more Google Ads campaigns, ad groups, ads or keywords in a single call.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `level` | string | yes | `campaigns`, `ad_groups`, `ads` or `keywords`. |
| `entityId` | string | one of entityId/entityIds | Bare id, the composite `{adGroupId}~{id}`, or a full resource name. |
| `entityIds` | array | one of entityId/entityIds | Several ids, same level, at most 100 total. |
| `status` | string | yes | `enabled`, `paused` or `removed`. |
| `confirm` | boolean | yes for `removed` | Removing is permanent and cannot be restored. |
| `validateOnly` | boolean | no | Preview what would happen without `confirm`. |

Kind: write, destructive. Scope: `ads:write`.

### `crm_update_google_ads_bid`

Set or clear a manual max CPC bid on an ad group or a keyword, in the account currency, not micros.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `level` | string | yes | `ad_group` or `keyword`. |
| `entityId` | string | yes | The ad group id, or the keyword's criterion id / composite `{adGroupId}~{criterionId}`. |
| `cpcBid` | number | one of cpcBid/clear | Greater than 0. Required for `ad_group`. |
| `clear` | boolean | no | Keyword level only: removes the keyword's own bid so it uses its ad group's bid again. |
| `validateOnly` | boolean | no | Checks the change with Google without applying it. |

Kind: write. Scope: `ads:write`.

### `crm_update_google_ads_bidding`

Switch a campaign's bidding strategy or change its current target, in the account currency with ratios as fractions.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | yes | Campaign to change. |
| `strategy` | string | yes | `manual_cpc`, `maximize_clicks`, `maximize_conversions`, `maximize_conversion_value`, `target_impression_share` or `portfolio`. |
| `targetCpa` | number | no | For `maximize_conversions` (optional). |
| `targetRoas` | number | no | For `maximize_conversion_value`, 3 means 300%. |
| `cpcBidCeiling` | number | no | For `maximize_clicks` or `target_impression_share`. |
| `impressionShareLocation` | string | for `target_impression_share` | `ANYWHERE_ON_PAGE`, `TOP_OF_PAGE` or `ABSOLUTE_TOP_OF_PAGE`. |
| `impressionShareTarget` | number | for `target_impression_share` | Fraction, e.g. 0.8 means 80%. |
| `portfolioStrategyId` | string | for `portfolio` | From `crm_google_ads_bidding_strategies`. |
| `clearTarget` | boolean | no | Removes the current limit or target instead of setting a new one. |

Kind: write. Scope: `ads:write`.

### `crm_update_google_ads_budget`

Set the daily budget of a Google Ads campaign, in the account currency, not micros. Takes effect immediately.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | yes | Campaign to change. |
| `dailyBudget` | number | yes | New daily budget, account currency. |
| `confirmShared` | boolean | yes if budget is shared | Required when the budget is shared with other campaigns; the call otherwise lists which campaigns share it. |

Kind: write, destructive. Scope: `ads:write`.

### `crm_update_google_ads_targeting`

Change a campaign's locations, languages, ad schedule or device bid modifiers; several kinds can change in one atomic call.

| Argument | Type | Required | Notes |
|---|---|---|---|
| `customerId` | string | yes | Google Ads customer id, digits only. |
| `campaignId` | string | yes | Campaign to change. |
| `locationOption` | string | no | `PRESENCE` or `PRESENCE_OR_INTEREST`, for included locations. |
| `excludedLocationOption` | string | no | `PRESENCE` or `PRESENCE_OR_INTEREST`, for excluded locations. |
| `addLocations` | array | no | A `geoTargetId` string, or `{geoTargetId, bidModifier}`. |
| `excludeLocations` | array | no | `geoTargetIds` to exclude. |
| `removeLocations` | array | no | Criterion ids or geoTargetIds to stop targeting or excluding. |
| `locationBidModifiers` | array | no | Changes the bid modifier of an already-included location. |
| `languages` | array | no | Replaces the whole language set; empty array means every language. |
| `addLanguages` | array | no | Incremental alternative to `languages`. |
| `removeLanguages` | array | no | Incremental alternative to `languages`. |
| `adSchedule` | array | no | Replaces the whole schedule, at most 6 windows per day; empty array means every hour. |
| `deviceModifiers` | object | no | `{mobile, desktop, tablet}`, each 0.1 to 10, or 0 to exclude the device. |
| `confirm` | boolean | yes to drop every location | Required when `removeLocations` would leave no included location. |
| `validateOnly` | boolean | no | Checks without applying. |

Kind: write. Scope: `ads:write`.
