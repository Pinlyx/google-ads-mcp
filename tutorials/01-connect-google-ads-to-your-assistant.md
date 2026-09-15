# Connect Google Ads to Your AI Assistant in 10 Minutes

By the end of this tutorial your assistant will answer "what did we spend on Google Ads last week,
and which campaigns did the spending" by calling the ad account directly, and you will have made
one guarded write and undone it. Four stages: connect the ad account, create a key, wire the
server into your client, make the calls. Every stage ends with a check that fails if you skipped
the one before it.

The worked example uses the CRM Solid MCP server, which wraps the Google Ads API and speaks to
your own account over OAuth. The shape is the same for any stdio MCP server: only the package
name, the environment variable and the tool names change.

## What you need

| Requirement | Why | Check |
|---|---|---|
| Node.js 20 or newer | The bridge is ESM and declares `node >= 20` | `node --version` |
| An MCP client | Claude Desktop, Claude Code and Cursor are covered below | Any recent build |
| A Google Ads account you can sign into | The OAuth consent is granted by its owner | You can open ads.google.com |
| A CRM Solid account | Holds the connection and the API key. The free plan is enough | Created in step 1 |
| `curl` | Verifies the server without a client in the way | `curl --version` |

Time: about 10 minutes, most of it waiting for a client to restart.

## Step 1: Connect your Google Ads account

Sign in at [app.crmsolid.com](https://app.crmsolid.com), then open **Insights > Ads > Google Ads**
and press **Connect Google Ads account**. Google asks you to sign in, may ask for a passkey or
other second factor, and then shows the consent screen listing one permission: see, edit, create
and delete your Google Ads accounts and data. That single permission is the only scope the Google
Ads API offers, which is why a read-only variant does not exist.

After you accept, the page lists every ad account that Google login can reach, and shows spend,
clicks, conversions and the campaign table for the one you select.

**Verify:** the page shows at least one account in the selector and a spend figure that matches
what you see in the Google Ads interface for the same date range. If the selector is empty, the
Google account you used has no ad account attached to it; sign in with the one that does.

## Step 2: Create an API key

Open [app.crmsolid.com/settings/developers](https://app.crmsolid.com/settings/developers) and
create a key. Grant it `ads:read` to start. Add `ads:write` only when you reach step 6 of this
tutorial, and prefer a second key for that rather than widening this one.

The value starts with `csk_live_` and is shown once. Store it where you keep other secrets. Scopes
are per key, so a key that can only read is a genuinely different key, not a setting you can flip
later.

**Verify:** the key list shows your new key with the scopes you granted and a prefix like
`csk_live_1a2b3c4d`.

## Step 3: Check the server answers, without a client

Before touching any client config, prove the key works. This is a JSON-RPC call over HTTP to the
hosted server.

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer csk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

The response is a long JSON document listing every tool the key may call. To see only the ads
tools:

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer csk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' \
  | grep -o '"name":"crm_[a-z_]*ads[a-z_]*"'
```

Expected output includes `crm_list_google_ads_accounts`, `crm_google_ads_summary`,
`crm_google_ads_campaigns`, `crm_google_ads_breakdown` and
`crm_google_ads_campaign_settings`.

**Verify:** you get tool names, not `401`. A `401` means the key is wrong, revoked, or was copied
with a trailing space.

## Step 4: Find your customer id

Every other tool needs the ad account's customer id. Ask the server:

```bash
curl -s https://api.crmsolid.com/mcp \
  -H "Authorization: Bearer csk_live_your_key_here" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call",
       "params":{"name":"crm_list_google_ads_accounts","arguments":{}}}'
```

The result contains one entry per connected account with `customerId`, `name`, `currency` and
`timeZone`, plus a `dailyQuota` block telling you how many requests you have left today.

```json
{
  "count": 1,
  "accounts": [
    { "customerId": "4953595856", "name": "Acme Ltd", "currency": "GBP", "timeZone": "Europe/London" }
  ],
  "dailyQuota": { "used": 0, "limit": 50, "remaining": 50, "unlimited": false }
}
```

Customer ids are digits only here. If you copy one out of the Google Ads interface it will look
like `495-359-5856`; drop the dashes.

**Verify:** you have a `customerId` value in hand. Keep it nearby, every call below uses it.

## Step 5: Wire the server into your client

The hosted server speaks HTTP. Clients that expect a local command use the npm bridge, which runs
on your machine, holds the key in its environment and forwards calls. `--tools ads` limits the
session to the Google Ads tools so the model is not distracted by the rest of the CRM.

### Claude Desktop

Edit `claude_desktop_config.json` (Settings > Developer > Edit Config):

```jsonc
{
  "mcpServers": {
    "crmsolid-ads": {
      "command": "npx",
      "args": ["-y", "@crmsolid/mcp-server", "--tools", "ads"],
      "env": { "CRMSOLID_API_KEY": "csk_live_your_key_here" }
    }
  }
}
```

Quit Claude Desktop completely and reopen it. On Windows and macOS, closing the window is not
quitting.

### Claude Code

```bash
claude mcp add crmsolid-ads \
  --env CRMSOLID_API_KEY=csk_live_your_key_here \
  -- npx -y @crmsolid/mcp-server --tools ads
```

### Cursor

Add the same `mcpServers` block to `.cursor/mcp.json` in your project, or to the global file in
Cursor settings, then reload the window.

**Verify:** the client lists the server as connected and shows the ads tools. In Claude Desktop
the tool count appears under the input box; in Claude Code, `/mcp` lists servers and their tools.
If the server shows as failed, run `npx -y @crmsolid/mcp-server --tools ads` in a terminal with the
environment variable set and read the first line of output; a wrong key fails immediately and
says so.

## Step 6: Ask for something real

Now use words instead of JSON. Good first questions:

- "List my Google Ads accounts and tell me how many requests I have left today."
- "For customer 4953595856, what did we spend in the last 7 days, and what were the clicks and
  conversions?"
- "Show me the campaigns for that account sorted by spend, with their status."
- "Which ad groups in the top campaign have zero conversions?"

Under the hood those are `crm_list_google_ads_accounts`, `crm_google_ads_summary`,
`crm_google_ads_campaigns` and `crm_google_ads_breakdown` with `level` set to `ad_groups`. Ask the
client to show tool calls if you want to watch the mapping.

**Verify:** the numbers match the Google Ads interface for the same window. Small differences in
the last hour are normal, Google's own reporting lags. A figure that is out by an order of
magnitude usually means you compared different date ranges, or an account with a different
currency.

## Step 7: One guarded write, then undo it

Writes need a key with `ads:write`. Create a second key with both scopes, restart the client with
that key, and pick a campaign that is already paused so nothing changes in the real world.

Ask: "Read the settings of campaign 24253282018 in account 7420720386, then pause it."

The assistant should call `crm_google_ads_campaign_settings` first, tell you the campaign is
already paused, and either stop or make a no-op call. Then ask it to enable the campaign, check
the Google Ads interface, and pause it again.

Two habits to build now:

- Ask for the settings read before any change. A budget or bidding change made without reading
  the current value is a guess.
- Keep `--read-only` in the config for sessions where you only want analysis:

```jsonc
"args": ["-y", "@crmsolid/mcp-server", "--tools", "ads", "--read-only"]
```

**Verify:** the campaign's status in the Google Ads interface followed your instructions, and
ended back where it started.

## What each tool costs you

The free plan allows 50 Google Ads requests a day, Pro 500, Business unlimited. Only calls that
reach Google count, and reports are cached for 15 minutes, so asking the same question twice in a
row costs one request, not two. A breakdown call returns up to 500 rows, so pulling every search
term for a week is a single request, not one per term.

When the allowance runs out the server answers HTTP 429 with a message naming the limit and
saying it resets at midnight UTC. Nothing breaks; the next day starts fresh.

## Where to go next

- [Find wasted Google Ads spend](./02-find-wasted-google-ads-spend.md), the search term loop.
- [Pause campaigns and move budget, safely](./03-pause-and-rebudget-safely.md), the guarded write
  pattern in full.
- [Scopes, quotas and safety](./06-scopes-quotas-and-safety.md) before you give any assistant a
  key with `ads:write`.
