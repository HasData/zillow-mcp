# Zillow MCP Server

<!-- mcp-name: com.hasdata/zillow -->

A hosted Model Context Protocol (MCP) server that gives Claude, Cursor, Windsurf and any other MCP client two read-only Zillow tools. Search for-sale, for-rent and sold listings with rich filters, and read a single property in full, all as structured JSON, with no Zillow account and nothing to host.

It reads public listing pages on Zillow.com that a signed-out visitor can see.

```
https://mcp.hasdata.com/api/mcp?apis=zillow
```

[![Glama score](https://glama.ai/mcp/servers/HasData/zillow-mcp/badges/score.svg)](https://glama.ai/mcp/servers/HasData/zillow-mcp)
[![tool contract](https://github.com/HasData/zillow-mcp/actions/workflows/contract.yml/badge.svg)](https://github.com/HasData/zillow-mcp/actions/workflows/contract.yml)
[![MCP](https://img.shields.io/badge/MCP-remote%20%7C%20streamable%20HTTP-6366f1?style=flat-square)](https://modelcontextprotocol.io)
[![Tools](https://img.shields.io/badge/tools-2-10b981?style=flat-square)](#tools)
[![License](https://img.shields.io/badge/license-MIT-blue?style=flat-square)](LICENSE)

## Contents

- [What you need](#what-you-need)
- [Quick start](#quick-start)
- [Example prompts](#example-prompts)
- [Tools](#tools)
- [Errors and failure paths](#errors-and-failure-paths)
- [Pricing, free tier and limits](#pricing-free-tier-and-limits)
- [Tool selection](#tool-selection)
- [How it compares](#how-it-compares)
- [FAQ](#faq)
- [HasData links](#hasdata-links)
- [Development](#development)
- [Contributing](#contributing)
- [License](#license)

## What you need

An MCP client and a HasData API key from the [dashboard](https://app.hasdata.com/sign-up?utm_source=github&utm_medium=syndication&utm_campaign=zillow-mcp), free to create with no card, and the trial covers about 200 calls at the 5-credit rate. This is a remote server, so the simplest path is a URL and an `x-api-key` header, with no container to run and no Zillow account anywhere in the flow. A client that only speaks stdio reaches it through a thin launcher, published as `@hasdata/zillow-mcp` on npm and `hasdata-zillow-mcp` on PyPI, shown below.

## Quick start

The server URL is the same for every client. We run it hands-on in Claude Code and Claude Desktop. The other blocks follow each client's own documented format for a remote server.

| Field | Value |
| :--- | :--- |
| URL | `https://mcp.hasdata.com/api/mcp?apis=zillow` |
| Transport | HTTP, streamable |
| Auth header | `x-api-key: HASDATA_API_KEY` |

Clients with OAuth support can add the same URL as a connector and sign in without putting a key in a config file.

<details>
<summary><b>Claude Code</b></summary>

```bash
claude mcp add --transport http zillow "https://mcp.hasdata.com/api/mcp?apis=zillow" \
  --header "x-api-key: HASDATA_API_KEY"
```

</details>

<details>
<summary><b>Claude Desktop</b></summary>

Settings, then Connectors, then Add custom connector, then paste `https://mcp.hasdata.com/api/mcp?apis=zillow` and sign in.

For the config-file route, Claude Desktop loads only local (stdio) servers, so it reaches a remote server through a stdio launcher. The `@hasdata/zillow-mcp` package is that launcher, and it reads the key from the environment. Add this to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "zillow": {
      "command": "npx",
      "args": ["-y", "@hasdata/zillow-mcp"],
      "env": { "HASDATA_API_KEY": "YOUR_KEY" }
    }
  }
}
```

For Python instead of Node, swap the launcher for the PyPI package, which `uvx` runs without a manual install:

```json
{
  "mcpServers": {
    "zillow": {
      "command": "uvx",
      "args": ["hasdata-zillow-mcp"],
      "env": { "HASDATA_API_KEY": "YOUR_KEY" }
    }
  }
}
```

</details>

<details>
<summary><b>Cursor</b></summary>

`~/.cursor/mcp.json` for every project, or `.cursor/mcp.json` for one:

```json
{
  "mcpServers": {
    "zillow": {
      "url": "https://mcp.hasdata.com/api/mcp?apis=zillow",
      "headers": { "x-api-key": "HASDATA_API_KEY" }
    }
  }
}
```

</details>

<details>
<summary><b>Windsurf</b></summary>

`~/.codeium/windsurf/mcp_config.json`. Windsurf calls the field `serverUrl`, not `url`:

```json
{
  "mcpServers": {
    "zillow": {
      "serverUrl": "https://mcp.hasdata.com/api/mcp?apis=zillow",
      "headers": { "x-api-key": "HASDATA_API_KEY" }
    }
  }
}
```

</details>

<details>
<summary><b>VS Code</b></summary>

`.vscode/mcp.json` in the workspace:

```json
{
  "servers": {
    "zillow": {
      "type": "http",
      "url": "https://mcp.hasdata.com/api/mcp?apis=zillow",
      "headers": { "x-api-key": "HASDATA_API_KEY" }
    }
  }
}
```

</details>

## Example prompts

Prompts, not code. Paste one in and the agent picks the tool itself. Each is annotated with the calls it takes, because every successful call costs 5 credits.

> Search for-sale homes in Austin, TX with at least three beds under $600k, sorted newest first, and give me the ten most recent with price and days on market.

*One call, 5 credits. Price, beds, area and days on market come back on the search result.*

> Take the top result and pull its full detail: price history, tax history, the price estimate, and the assigned schools.

*One call, 5 credits. Those live on the property page, which the details tool reads by URL.*

> Find for-rent condos in Austin that allow cats, then pull the rental estimate on the three cheapest.

*Four calls, 20 credits. One search, then one property call for each of the three.*

> For this property URL, give me the list price, the price estimate, and the last three sales in its price history.

*One call, 5 credits.*

A search result is enough to rank and shortlist. Price history, tax history, the estimate, schools and the agent come from the property call, so a prompt that shortlists then inspects three homes is one search plus three property calls.

## Tools

Two tools, read-only. Samples below are trimmed from real calls, and the numbers move as the market moves. Read them as shapes. Each tool name links to its endpoint reference, which carries the full field list.

The samples are the payload, not the whole response. A `tools/call` result carries one text block, and that text is itself JSON holding `url`, `status`, `text` and `json`, with the scraped data under `json`. From a raw JSON-RPC response the path is `result.content[0].text`, parsed, then `.json`. A chat client unwraps that for you and code talking to the endpoint directly does not.

### Get Zillow real estate listings

[`hasdata_zillow_listing_getRealEstateListings`](https://docs.hasdata.com/apis/zillow/listing?utm_source=github&utm_medium=syndication&utm_campaign=zillow-mcp)

A page of listings by location keyword, filtered.

| Parameter | Type | Required | Notes |
| :--- | :--- | :--- | :--- |
| `keyword` | string | yes | The location to search, such as `Austin, TX` |
| `type` | string | yes | `forSale`, `forRent` or `sold` |
| `price_min_` / `price_max_` | number | | Price band |
| `beds_min_` / `beds_max_` / `baths_min_` / `baths_max_` | number | | Bed and bath bands |
| `homeTypes__` | array | | `house`, `condo`, `townhome`, `multiFamily`, `apartment`, `lot`, `manufactured` |
| `daysOnZillow` | number/string | | `1`, `7`, `14`, `30`, `90`, `6m`, `12m` and up |
| `sort` | string | | `newest`, `priceLowToHigh`, `priceHighToLow`, `squareFeet` and more |
| `page` | number | | Result page |

The reference also documents square-footage, lot-size, year-built and HOA bands, plus `otherAmenities__`, `views__`, `pets__`, `listingType`, `propertyStatus__`, `listingPublishOptions__` and more.

Returns `searchInformation` with `totalResults`, a `properties` array, and `pagination` whose `nextPage` is the URL of the following page. Each property carries `id`, `url`, `homeType`, `status`, `price`, `currency`, a `rentZestimate` rental estimate, `daysOnZillow`, `area` in square feet, `addressRaw` and a structured `address`, `latitude`, `longitude`, `beds`, `baths`, `listingDetails`, `mediaDetails` and `photos`.

```json
{
  "id": "60134551",
  "url": "https://www.zillow.com/homedetails/6116-Speyside-Dr-Austin-TX-78754/60134551_zpid/",
  "homeType": "SINGLE_FAMILY",
  "status": "FOR_SALE",
  "price": 320000,
  "currency": "$",
  "rentZestimate": 2286,
  "daysOnZillow": 0,
  "area": 2277,
  "address": { "street": "6116 Speyside Dr", "city": "Austin", "state": "TX", "zipcode": "78754" },
  "beds": 4,
  "baths": 3
}
```

### Get Zillow property details

[`hasdata_zillow_property_getPropertyDetails`](https://docs.hasdata.com/apis/zillow/property?utm_source=github&utm_medium=syndication&utm_campaign=zillow-mcp)

One property in full, by its URL.

| Parameter | Type | Required | Notes |
| :--- | :--- | :--- | :--- |
| `url` | string | yes | A Zillow property URL, the `url` field from a listing result |
| `extractAgentEmails` | boolean | | Attempt to pull the listing agent's email. Raises the credit cost of the call |

Returns the full page: `price`, `currency`, `fees`, `beds`, `baths`, `area`, `yearBuilt`, `homeType`, `mlsId`, a structured `address` and `geo`, the `description` and `highlights`, `photos`, `schools`, `daysOnZillow`, `views`, `saves`, an `agentInfo` block, and `priceHistory`, `taxHistory` and `mortgage` arrays. Zillow's own price estimate arrives in a `zestimate` object holding `zestimate`, an `estimatedSaleRange` and a `rentZestimate`. Read it as an estimate, not as a confirmed value.

```json
{
  "id": 60134551,
  "status": "FOR_SALE",
  "price": 320000,
  "currency": "USD",
  "yearBuilt": 2002,
  "beds": 4,
  "baths": 3,
  "area": { "livingArea": 2277, "livingAreaUnits": "Square Feet" },
  "fees": { "monthlyHoaFee": "$500 annually" },
  "zestimate": { "zestimate": 318100, "estimatedSaleRange": "$302K - $334K", "rentZestimate": 2286 },
  "address": { "street": "6116 Speyside Dr", "county": "Travis County" },
  "agentInfo": { "agentName": "Marie Coleman", "brokerName": "eXp Realty" },
  "priceHistory": [{ "date": "2026-08-24", "price": 320000, "event": "listedForSale" }],
  "schools": { "elementarySchool": { "name": "Bluebonnet Trail", "district": "Manor ISD" } }
}
```

## Errors and failure paths

Your client almost never sees an HTTP error code from a tool call. The MCP layer answers 200 and puts the failure inside the result, with `isError` set to `true` and the reason as text. The agent reads a message where you might expect a status line.

**A wrong key surfaces as tool output, not as a failed connection.** `tools/list` accepts any non-empty key and returns both tools, so the client completes its handshake and shows green. The first tool call then comes back with `isError: true` and the text `HasData API error: 401 Unauthorized`. Watch for that string, because nothing earlier in the flow reports the problem.

**A missing key is the one real HTTP error.** Authorization runs before any tool, and the connection itself fails with 401. CORS headers are present, and a browser client reads the status and not an opaque network failure.

**An argument that breaks a tool's schema is rejected before it becomes a scrape.** The server answers with `isError: true` and the text `MCP error -32602: Input validation error`, naming the offending field. Nothing is fetched and nothing is charged.

**A search with no matches returns a successful result with an empty `properties` array**, not an error. A location and filter set with no inventory still comes back with `requestMetadata.status` set to `ok`. Test for the array length before you iterate.

**A property that has been delisted returns 400** with `requestMetadata.status` set to `error`. A URL from an old search can point at a listing that is gone.

Results that carry data also carry a `requestMetadata.id` worth quoting in support.

## Pricing, free tier and limits

Each Zillow tool costs **5 credits per successful call**. Turning on `extractAgentEmails` raises the property call's cost, so leave it off unless you need the email. Response size does not change the price.

The free trial is **1,000 credits over 30 days with no card**, which is 200 Zillow calls at the base rate. After that an active account keeps getting 100 credits topped up each day whenever its balance drops below 100, so a low-volume agent runs on the free tier indefinitely.

Paid plans start at **$49 a month** for 200,000 credits, which is 40,000 calls. The unit price falls with volume, from **$1.23 per 1,000 calls** on the entry plan to **$0.50** on Business, **$0.42** on Growth and **$0.37** on the largest [high-volume plans](https://hasdata.com/prices?utm_source=github&utm_medium=syndication&utm_campaign=zillow-mcp).

Your plan also sets concurrency. The free trial allows 1 request at a time, Startup 15, Business 30, Growth 50, and the high-volume plans run from 200 to 1,500. Handle the overflow case defensively in anything unattended.

A request that comes back non-200 is not billed. A successful call that finds nothing is still a call.

## Tool selection

The `apis` query parameter decides which tools your agent sees. Fewer tools means less context spent on tool definitions, and fewer chances for the model to reach for the wrong one.

```
?apis=zillow                     the two tools in this repo
?apis=zillow,redfin              add Redfin real estate
?apis=zillow,google_maps         add Google Maps places
```

The parameter takes provider names like `zillow` and individual API names like `zillow_listing`. Misspelled names are ignored. If every name is wrong the request fails with 400, and the body lists both what it did not recognise and every valid value. Drop the parameter and the same endpoint exposes all 57 HasData tools.

## How it compares

Zillow's own API programs are for members and partners moving their own inventory, such as the Bridge Interactive and Mortgage APIs, not a self-serve way to read the public market. For searching listings and reading arbitrary properties, scraping the public pages is the route, and this server does that behind a stable schema.

| | Zillow partner APIs | This server |
| :--- | :--- | :--- |
| Purpose | Move your own or MLS inventory | Read the public market |
| Access | Membership or partner approval | One key and one URL |
| Search across the market | Restricted | Yes, with rich filters |
| Setup | Business onboarding | None |
| Output | Partner feeds | Structured JSON, price and beds pre-parsed |

**What this server does not do.** No posting, no lead submission, no account data. It reads what a signed-out visitor can see on Zillow.com.

## FAQ

### Is there an official Zillow MCP server?

Zillow does not publish one. This one is maintained by HasData and reads public pages, which is why it needs no Zillow account.

### What is a Zillow MCP server?

A server that exposes Zillow listing data as tools an AI client can call. The client sends a tool call over the Model Context Protocol, the server fetches the data and returns structured JSON, and the model works with the result. This one exposes two tools and runs remotely.

### Do I need a Zillow account or API key?

No. The only credential is your HasData key. There is no Zillow membership to apply for, because the tools read public Zillow.com pages.

### Why does a search result not show price history or schools?

Because Zillow does not put them on the search card. They live on the property page, which the details tool reads by URL. Search to shortlist, then call the property tool for depth.

### What does the price estimate mean?

The `zestimate` object holds Zillow's own estimated value, a range around it, and a rental estimate. It is a model output, not an appraisal or a confirmed sale price. Treat it as an estimate.

### Can I use this together with other HasData APIs?

Yes. The `apis` parameter takes a list, and `?apis=zillow,redfin` gives your agent Zillow plus Redfin. [Drop the parameter](#tool-selection) and you get everything.

### Is HasData affiliated with Zillow?

No. HasData is an independent service and is not affiliated with, endorsed by, or sponsored by Zillow Group, Inc. Zillow is a trademark of its respective owner.

### Compliance and personal data

HasData accesses publicly available data only. A platform's terms may restrict automated access, and you are responsible for your own compliance. Where the data you collect includes personal information, such as a listing agent's contact details, make sure you have a lawful basis for it under GDPR, CCPA or the equivalent rules in your jurisdiction.

## HasData links

| | |
| :--- | :--- |
| Product page and request builder | [Zillow Scraper API](https://hasdata.com/apis/zillow-api?utm_source=github&utm_medium=syndication&utm_campaign=zillow-mcp) |
| Server documentation | [MCP server docs](https://docs.hasdata.com/mcp-server?utm_source=github&utm_medium=syndication&utm_campaign=zillow-mcp) |
| All 57 tools in one server | [HasData/hasdata-mcp](https://github.com/HasData/hasdata-mcp) |
| Client walkthroughs | [MCP clients and integrations](https://hasdata.com/integrations/mcp?utm_source=github&utm_medium=syndication&utm_campaign=zillow-mcp) |
| Everything else we scrape | [Zillow Scraper API and 54 more](https://hasdata.com/apis/?utm_source=github&utm_medium=syndication&utm_campaign=zillow-mcp) |
| Plans and credit costs | [Plans and credit costs](https://hasdata.com/prices?utm_source=github&utm_medium=syndication&utm_campaign=zillow-mcp) |
| Keys and usage | [HasData dashboard](https://app.hasdata.com?utm_source=github&utm_medium=syndication&utm_campaign=zillow-mcp) |

## Development

This repository is configuration and documentation for a remote server. There is no build step and nothing to containerize.

The tests in `test/` assert the tool contract, the part that can break without a commit here. They check that `?apis=zillow` returns exactly two tools, that every tool still declares its required parameters, that no name changed, and that the key in use is actually accepted. That last check calls a tool for real and costs 5 credits, which is the price of a canary that can fail for the right reason.

```bash
# macOS and Linux
HASDATA_API_KEY=your_key_here npm test

# Windows PowerShell
$env:HASDATA_API_KEY="your_key_here"; npm test
```

The same suite runs in CI on every push and once a week on a schedule, because the upstream tool list can change without anyone touching this repository. A failure means the tool list moved, the key stopped working, or the endpoint was unreachable, and the assertion message says which.

## Contributing

Corrections to the tool tables and the response samples are the most useful contribution, because those are the parts that drift. Include the call you made and the response you got. Pull requests from forks run the suite without a key, and the live checks skip instead of going red.

## License

MIT. See [LICENSE](LICENSE).
