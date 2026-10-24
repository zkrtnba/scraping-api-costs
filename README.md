# web scraping n8n template: what the popular ones skip, how to plug in a scraping API, and what each request really costs

Most n8n web scraping templates you'll find work great for about five minutes. They fetch a page, parse it, drop rows into Google Sheets, and everything is fine until the target site decides your n8n server's IP looks like a bot. Then the workflow starts returning Cloudflare challenge pages, and you're debugging HTML instead of collecting data.

That's the real gap behind this search. The templates themselves are usually well built; the fragile part is the raw HTTP request sitting in the middle of the workflow. A common fix is routing the scraping step through an API that handles proxies, browser rendering, and CAPTCHA bypass for you — and ScraperAPI has an official n8n node plus pre-built templates for exactly this.

**Quick picks**

> Most people searching for an n8n scraping template should start with ScraperAPI's free tier (1,000 credits/month, 5 concurrent connections) or the 7-day trial with 5,000 credits and no credit card. Paid plans start at $49/month for 100,000 credits, and annual billing takes 10% off across the board. If your workflows touch Amazon, Google, or anything behind Cloudflare, the per-request cost details below matter more than the sticker price.
>
> 👉 [Start the 5,000-credit trial and grab your API key](https://www.scraperapi.com/signup?fp_ref=coupons)

## What the popular n8n scraping templates actually do

The templates ranking for this search fall into a few clear buckets, and knowing which bucket you're in saves a lot of wasted setup time.

The most typical style is the configurable multi-page scraper: you set a start URL, a pagination rule, and a CSS selector, and the workflow loops through pages collecting structured data. These are genuinely good starting points if your target site has no bot protection — think internal tools, public listings, or sites you control.

The second bucket is service-based templates. Apify's templates run an Apify actor on a schedule, save new results to Google Sheets, and push matches to Telegram or Slack. Bright Data's approach pairs its scraping infrastructure with Google Gemini through an MCP client node, with the caveat that it only works on self-hosted n8n because it depends on a community node.

A third pattern adds resilience directly into the workflow: one template runs a standard scrape first and falls back to the Scrape.do service if the request hits anti-bot errors like timeouts. It's a sensible design, but it means paying for two services and maintaining two configurations.

Two things almost none of these template pages tell you: what each run actually costs once you're on a metered scraping API, and which node parameters quietly conflict with each other. Those two gaps are where most of the surprise bills and failed workflows come from, so they're covered in detail below.

## The official ScraperAPI node for n8n: setup and what it covers

ScraperAPI ships a first-party community node (n8n-nodes-scraperapi-official), which means you don't need to hand-build HTTP Request nodes with query strings and pray the encoding is right.

Setup takes three steps:

1. In n8n, open the Nodes panel, search for **ScraperAPI**, and click Install.
2. Copy your API key from the ScraperAPI dashboard.
3. In a workflow, add the ScraperAPI node, click "Credential to connect with," create a new credential, paste the key, and save. The credential auto-tests on save.

The node exposes four resources, and this is worth understanding before you pick a template:

- **API** — scrape any single URL. Supports JavaScript rendering, geo-targeting by two-letter country code, desktop/mobile user agents, premium and ultra-premium proxies, and markdown/text output (JSON and CSV are available for autoparse-supported sites).
- **Structured Data Endpoints** — pre-built scrapers for Amazon (Product, Search, Offers), Google (Search, Jobs, News, Shopping, Maps), eBay (Search, Product), Walmart (Search, Category, Product, Review), and Redfin (For Sale, For Rent, Search, Agent). These return clean parsed JSON instead of raw HTML, which usually removes several Parse nodes from your workflow.
- **Crawler** — multi-page crawl jobs with start URL, max depth, crawl budget, and URL regex filters. One thing to know before you build on it: the crawler returns a `jobId` and streams results to your webhook rather than outputting them at the node itself, so your workflow structure needs a webhook receiver.
- **AI Parser** — build a reusable parser from a few example URLs, then apply it to any page with the same layout. Useful for sites that have no structured data endpoint.

If you'd rather not assemble this yourself, ScraperAPI maintains official templates on the n8n library, including daily price-drop digests for Amazon, Walmart, and Google that you can import and adapt.

For AI-agent workflows, the documented pattern is: a Chat Message Received trigger → an AI Agent node → an OpenAI (or other) chat model on the Chat Model input → a Simple Memory node → the ScraperAPI node on the Tool input, plus a system prompt. There's also a hosted MCP server at `https://mcp.scraperapi.com/mcp` (HTTP Streamable transport, your API key as the Bearer token) if you prefer wiring scraping in through n8n's MCP Client Tool.

Once the node is in place, your "template" problem mostly reduces to choosing the right parameters — and knowing what each one costs.

## What a request actually costs: the credit math most template pages skip

ScraperAPI meters everything in API credits, and the flat "1 credit = 1 request" assumption is wrong for most interesting targets. Per the current documentation:

| Request type | Credits per request |
| --- | --- |
| Standard page | 1 |
| Amazon, Walmart, eBay | 5 |
| Google, Bing (all subdomains) | 25 |
| LinkedIn | 30 |
| Anti-bot bypass (Cloudflare, Cloudflare Turnstile, Datadome, PerimeterX, Akamai) | +10 each |

Parameters stack on top. `render=true` (JavaScript rendering) adds 10 credits, `premium=true` adds 10, `screenshot=true` adds 10, and `ultra_premium=true` adds 30 (paid plans only). The expensive combos: premium + render is 25 credits per request, and ultra_premium + render is 75.

Run the numbers on the Hobby plan's 100,000 credits and the differences get concrete. Plain pages: 100,000 requests. Amazon product pages with rendering: 15 credits each (5 base + 10 render), so about 6,600 requests. A Google SERP scrape with premium proxies and rendering: 45 credits per request, roughly 2,200 requests from the same balance. Same plan, three very different capacities.

Two mitigating details the pricing page spells out: the `autoparse`, `country_code`, `session_number`, `device_type`, `output_format`, and `keep_headers` parameters add no cost, and every response includes a `sa-credit-cost` header showing exactly what that request consumed. Before committing a workflow to a schedule, you can check a URL's cost in advance — via the Domain Multiplier in the dashboard, the API Playground, or a plain GET to `https://api.scraperapi.com/account/urlcost?api_key=API_KEY&url=...` with all your intended parameters attached.

One billing rule worth committing to memory: ScraperAPI only charges for successful requests — status codes 200 and 404 — plus requests you cancel yourself within 70 seconds. Failed requests blocked by the target site don't drain your balance, which materially changes the economics on hostile sites.

If the metering makes you nervous, every plan includes a `max_cost` setting that caps what any single request can spend. You can 👉 [check current plan sizes and pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) against your own cost estimates before committing.

## Parameter conflicts that break workflows silently

This is the second blind spot in nearly every template write-up: several ScraperAPI parameters can't be combined, and the node won't always fail loudly when you try.

The documented incompatibilities:

- `premium` and `ultraPremium` are mutually exclusive.
- `sessionNumber` (reusing the same proxy across requests) doesn't work with either `premium` or `ultraPremium`.
- `retry404` is unavailable with premium or ultra-premium proxies.
- `keepHeaders` and `customHeaders` are discarded when `ultraPremium` is enabled — and the docs add a candid warning that overriding ScraperAPI's own header management often lowers your success rate anyway.
- `desktopDevice` and `mobileDevice` can't both be set.
- `instructionSet` (browser commands like `click` or `wait_for_event`) and `waitForSelector` both implicitly enable rendering, which means they also implicitly add the render cost.

The practical takeaway: if a scheduled workflow starts failing after you "upgraded" it to premium proxies, check whether you left a session number in the config. These conflicts are all listed in the node's GitHub README, which is a better reference than any template screenshot.

## Plans and pricing: the full table

Here's everything currently listed on the pricing page. Annual billing runs 10% cheaper on every tier, and the feature set (JS rendering, premium residential and mobile IPs, CAPTCHA handling, structured data APIs, DataPipeline, crawler access) is identical across paid plans — the differences are credits, concurrency, and geo-targeting granularity.

| Plan | Monthly | Annual (per month) | Credits/month | Concurrency | Get started |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000 | 5 | [Sign up for the free plan](https://www.scraperapi.com/signup?fp_ref=coupons) |
| Hobby | $49 | $44.10 | 100,000 | 20 | [View Hobby plan pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | $149 | $134.10 | 1,000,000 | 50 | [View Startup plan pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | $299 | $269.10 | 3,000,000 | 100 | [View Business plan pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | $475 | $427.50 | 5,000,000 | 200 | [View Scaling plan pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | $975 | $877.50 | 10,500,000 | 300 | [View Professional plan pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 | [View Advanced plan pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ | [Talk to sales about Enterprise](https://www.scraperapi.com/?fp_ref=coupons) |

Note the free plan and the trial are separate: signing up gets you 1,000 credits per month indefinitely, while the 7-day trial carries a one-time 5,000 credits.

The finer print that affects plan choice:

- **Geo-targeting**: Hobby and Startup are limited to US and EU locations. Country-level targeting starts at Business.
- **Analytics**: Hobby retains data for 30 days; Business and up get unlimited analytics history.
- **Concurrency**: hitting the concurrent-request cap returns a 429 and the request is rejected rather than queued. There's no overage fee, but a workflow that fans out heavily may need the retry logic in n8n rather than at the API level.

Notably, the only meaningful differences between a $49 plan and a $1,975 plan are volume and limits — every tier gets the same proxy quality, bypass tech, and crawler access. That makes downgrade pressure low; you're buying capacity, not features.

## When your credits run out mid-workflow

By default, scraping pauses until credits renew — your n8n workflow doesn't fail, it just stops getting data until the next cycle. That's usually the wrong behavior for scheduled jobs, so two opt-in mechanisms exist:

- **Auto-upgrade** (Hobby through Professional): when credits hit zero, your account automatically moves to the next plan up, which costs more per month but typically lowers the per-credit rate. You have to enable it; it's not on by default.
- **Pay-as-you-go** (Scaling through Enterprise): keep scraping past your allowance at a fixed per-credit rate, up to a monthly spending cap you set yourself.

Annual subscribers can also renew early to get a fresh batch of credits immediately.

For n8n users specifically, auto-upgrade is the safer choice on low tiers — a price-monitoring workflow that silently pauses for a week is worse than one that costs a bit more. If you enable pay-as-you-go, set that monthly cap before you schedule anything, since the whole point of the cap is to bound the damage of a runaway loop. 👉 [Start Hobby with a 10% annual discount](https://www.scraperapi.com/pricing?fp_ref=coupons) if that sounds like your setup.

## Quick answers

**Will an n8n scraping template work on a Cloudflare-protected site?** Only if the fetching layer can handle the challenge. Plain HTTP Request nodes in n8n will typically receive the challenge page instead of content, which is why the ScraperAPI node (or the documented bypass parameters) is the load-bearing part of the workflow.

**How many Amazon scrapes fit in the free plan?** At 5 credits per Amazon request, the free plan's 1,000 credits cover about 200 Amazon product pages per month — workable for a daily price check on a handful of products, tight for anything broader.

**Do failed scrapes cost credits?** No. Only 200 and 404 responses are billed, along with requests you cancel within 70 seconds. Blocked or timed-out attempts come free.

**Can I control where a scrape appears to come from?** Yes, via the `countryCode` parameter at no extra cost — though Hobby and Startup are limited to US and EU codes.

**Is there a trial before paying?** A 7-day trial with 5,000 credits, no credit card required, on top of the always-free 1,000-credit plan.

## Which route makes sense for you

Match your situation to a starting point:

- **Just experimenting with n8n scraping on unprotected sites**: the free plan and a community template are genuinely enough. Don't pay yet.
- **Scraping Amazon, Walmart, or Google SERPs on a schedule**: start Hobby at $49 (or the trial first), and budget by the credit table above — remember Google requests burn 25 credits each, not 1.
- **Targets behind Cloudflare, Datadome, or PerimeterX**: you'll need either `premium` (+10 credits) or `ultra_premium` (+30, paid plans only). Factor that into your plan size before subscribing, because it's the difference between 100,000 and 2,500 effective requests.
- **Large-scale or recurring data pipelines**: Business ($299) is where country-level geo-targeting and unlimited analytics begin, and it's also the tier where the 3M credits comfortably absorb premium+render combos.

Before you commit to any paid tier, run your two or three most important target URLs through the trial and check the `sa-credit-cost` header on each — that one number tells you more about your real monthly bill than any pricing table. When you're ready, 👉 [compare current plans and start your trial](https://www.scraperapi.com/pricing?fp_ref=coupons).
