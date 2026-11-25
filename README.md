# Web Scraping API Python: How to Call ScraperAPI From a Script, What Each Credit Actually Buys, and Which Plan Fits Your Crawl (Full Plan Table)

ScraperAPI is one HTTP endpoint, so the shortest Python path is a `requests.get` with two parameters: your API key and the URL you want. There's no SDK to install, no client object to configure, and nothing that breaks when you upgrade Python. A 7-day trial gives you 5,000 credits with no credit card, and a free plan keeps 1,000 credits a month at 5 concurrent threads after that.

ScraperAPI doesn't price by request count. It prices by credits, and one page doesn't always cost one credit — a JS-rendered page costs 10, a Google SERP costs 25. That single fact decides which plan you need far more than the sticker price does, so most of this article is about the real per-request cost and the code that produces it.

> **Most people should start on Hobby: 100,000 credits a month, 20 concurrent threads.** That's 100,000 plain HTML pages, or 10,000 pages with JavaScript rendering turned on, or 4,000 Google SERPs. It's the smallest paid tier that leaves headroom for retries and failed requests. If most of your targets need rendering or you're pulling SERPs, skip to Startup.
> 👉 [Check current Hobby pricing and start the 5,000-credit trial](https://www.scraperapi.com/pricing?fp_ref=coupons)

## All ScraperAPI plans side by side

Prices per tier are quoted on the pricing page and scale with the credit allowance, so check the live figure before checkout. Annual billing takes 10% off every tier from Hobby through Advanced; Enterprise is quoted per deal.

| Plan | Credits / month | Concurrent threads | Geotargeting | Analytics history | Purchase |
| --- | --- | --- | --- | --- | --- |
| Free | 1,000 | 5 | US & EU | — | [Start free, no card](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | 100,000 | 20 | US & EU | Last 30 days | [See Hobby pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | US & EU | Last 30 days | [See Startup pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | 3,000,000 | 100 | Country-level | Unlimited | [See Business pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | Country-level | Unlimited | [See Scaling pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | Country-level | Unlimited | [See Professional pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | Country-level | Unlimited | [See Advanced pricing](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Country-level | Unlimited | [Talk to sales about Enterprise volume](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Two rows in that table matter more than they look. Geotargeting stops at US & EU on Free, Hobby and Startup, so if you need country-level proxies in Brazil or Japan, Business is the floor. And analytics history caps at 30 days on the two cheapest paid tiers, which matters if you're debugging a scraper that broke three weeks ago.

## What one credit actually buys

Every response carries an `sa-credit-cost` header telling you what that request cost. Before you send it, the Domain Cost Estimator in the dashboard prices any URL, and `max_cost` caps what a single request is allowed to consume.

| Request type | Credits | Hobby (100k) | Startup (1M) | Business (3M) | Scaling (5M) |
| --- | --- | --- | --- | --- | --- |
| Plain HTML page | 1 | 100,000 | 1,000,000 | 3,000,000 | 5,000,000 |
| Amazon / Walmart / eBay | 5 | 20,000 | 200,000 | 600,000 | 1,000,000 |
| JS rendering (`render=true`) | 10 | 10,000 | 100,000 | 300,000 | 500,000 |
| Premium proxy + rendering | 25 | 4,000 | 40,000 | 120,000 | 200,000 |
| Google / Bing SERP | 25 | 4,000 | 40,000 | 120,000 | 200,000 |
| LinkedIn | 30 | 3,333 | 33,333 | 100,000 | 166,666 |
| Ultra premium + rendering | 75 | 1,333 | 13,333 | 40,000 | 66,666 |

Three things in that table surprise people who budget by plan size alone.

**Combining features costs more than the sum.** Premium proxies are 10 credits and rendering is 10, but premium plus rendering bills at 25, not 20. Ultra premium is 30 and rendering is 10, but the pair bills at 75. If your targets are both bot-protected and JavaScript-heavy, your effective request count is a quarter of what the plan name suggests.

**A screenshot is not a separate line item.** `screenshot=true` switches JavaScript rendering on automatically and bills 10 credits total, so it costs the same as a rendered page rather than 20.

**The parameters that cost nothing are the ones that save you work.** Geotargeting, sticky sessions, device type, output format and auto-parsing all cost zero extra credits. Auto-parsing in particular returns structured JSON from supported sites without an upcharge, which is rare in this category.

For scale: 5,000 trial credits is 5,000 plain pages, but only 500 rendered pages or 200 SERP pulls. Turn rendering on during the trial and your headroom drops by a factor of ten.

## How to call it from Python

Install nothing ScraperAPI-specific. The endpoint takes `api_key` and `url` as query parameters, and everything else is optional.

python
import requests

API_KEY = "YOUR_API_KEY"

response = requests.get(
    "https://api.scraperapi.com",
    params={
        "api_key": API_KEY,
        "url": "https://example.com/product/123",
        "max_cost": 1,
    },
    timeout=60,
)
response.raise_for_status()

html = response.text
print(response.headers.get("sa-credit-cost"), len(html))


Set `max_cost` low on the first request of a new target. If the site turns out to be an expensive one, the call fails instead of quietly burning 25 credits, and the Domain Cost Estimator tells you what it should cost.

Rendering, geotargeting and sessions are the same call with more parameters:

python
params = {
    "api_key": API_KEY,
    "url": "https://example.com/checkout",
    "render": "true",        # 10 credits instead of 1
    "country_code": "de",    # no extra cost
    "session_number": "42",  # no extra cost, reuses the same proxy
}
r = requests.get("https://api.scraperapi.com", params=params, timeout=90)


`render` is opt-in, which is the opposite of how some competing SDKs behave — ScrapingBee's Python client has `render_js` defaulting to `True`, so a lazy request there costs more than a lazy request here. `session_number` keeps the same exit IP across requests for multi-step flows, but it can't be combined with `premium` or `ultra_premium`, and those two can't be combined with each other.

Sending your own headers works without an upcharge, as long as you flag it:

python
import time

def fetch(url, tries=3):
    params = {
        "api_key": API_KEY,
        "url": url,
        "keep_headers": "true",
        "device_type": "mobile",   # no extra cost
    }
    r = None
    for attempt in range(tries):
        r = requests.get(
            "https://api.scraperapi.com",
            params=params,
            headers={"Accept-Language": "en-US,en;q=0.9"},
            timeout=70,
        )
        if r.status_code == 429 or r.status_code >= 500:
            time.sleep(2 ** attempt)
            continue
        return r
    return r


That retry loop handles the two failures you'll actually see. A 429 means you've exceeded your concurrency, not your credits: the request is refused rather than queued, and you aren't charged for it, so backoff-and-retry is the correct response. A 5xx is a target-side or proxy-side failure and deserves the same treatment.

If you'd rather not put the key in a query string, `api_key` and flags like `render` can be sent as request headers instead. Proxy mode is also available, where credentials are passed to the proxy connection itself rather than as headers.

## Billing rules that decide your plan, not your code

**Nothing rolls over.** Unused credits expire at renewal on every tier, including the annual ones. Annual plans can renew early to grab a fresh batch, but you can't bank a quiet month. Size down rather than over-buying "just in case."

**Running out behaves differently depending on where you are.** On Hobby, Startup and Business, scraping pauses until credits renew unless you've opted into auto-upgrade, which promotes you to the next tier when the allowance is gone. Pay-as-you-go — the option to keep scraping at a fixed per-credit rate up to a monthly cap you set — only exists on Scaling, Professional, Advanced and Enterprise. There's a real gap here: on the three cheapest paid tiers, your only ways past the limit are waiting or jumping a full tier, and tier jumps are large (100,000 to 1,000,000 credits between Hobby and Startup).

**Annual billing saves 10%, roughly 1.2 months of a 12-month commitment.** That's real money on Professional and Advanced, and less interesting on Hobby, where you're also locking in a credit allowance you may outgrow before the year ends.

**Refunds are 7 days, no questions asked.** Combined with the card-free trial, you can validate a target list before any money moves.
👉 [Start with the free trial before committing to a paid tier](https://www.scraperapi.com/pricing?fp_ref=coupons)

## Who shouldn't buy this

If you scrape a few hundred static pages a month, the free tier's 1,000 credits and 5 threads cover you and paying is waste.

If you already run Playwright against one target site with your own proxy pool, do the arithmetic first: at 10 credits per rendered page, 100,000 credits is 10,000 rendered pages a month. If your volume is well under that and maintenance isn't hurting you, the subscription buys convenience, not capability.

If what you want is an AI-driven extraction API rather than raw HTML and JSON — Firecrawl's Python SDK, for instance, returns `.markdown` directly from `app.scrape()` and lets you drive a live browser session with natural-language prompts through `app.interact()` — ScraperAPI's parameter-based model will feel like manual labour. ScraperAPI gives you the page; deciding what's on it is your code's job, with auto-parsing handling the supported sites.

The opposite case is also worth naming. If you want zero dependencies beyond `requests`, no SDK version to track, and the same HTTP call from Python, Go, PHP or a shell script, ScraperAPI's model is the simplest of the group. Scrapeless, by comparison, exposes a service-namespace client (`client.browser`, `client.universal`, `client.scraper`, `client.deepserp`) built around actor strings, and its AI Scraper requires you to poll `get_task_result` yourself because the SDK doesn't do it for you. More structure, more to learn.

## FAQ

**Does ScraperAPI have an official Python SDK?**
No. You call the HTTP endpoint with `requests` or `httpx`. That's the intended integration path, not a workaround.

**Is the free trial enough to evaluate it?**
5,000 credits over 7 days with no credit card is generous for plain HTML and thin for anything else. Budget it as 5,000 pages, 1,000 Amazon pages, 500 rendered pages, or 200 SERPs, and test against the expensive targets first.

**What happens when credits run out?**
Scraping pauses until renewal. On Scaling and above you can enable pay-as-you-go with a monthly cap; on Hobby through Professional you can opt into auto-upgrade. On Advanced and Enterprise auto-upgrade isn't offered.

**Why am I getting 429s when I have credits left?**
You're over your concurrent-thread limit, not out of credits. There's no surcharge for hitting it and the request isn't queued, so retry with exponential backoff or reduce your worker count.

**Can I get raw HTML without paying for rendering?**
Yes. `render` is off unless you set it, and a plain page costs 1 credit. You only pay the 10-credit rendering fee on pages that actually need JavaScript to produce content.

**How do I know the cost before I scrape?**
Use the Domain Cost Estimator in the dashboard for a specific URL, set `max_cost` per request, and read `sa-credit-cost` from the response header afterwards. The three together let you log real cost per target rather than estimating it.

## Which plan to pick

Match the plan to your request type, not your page count:

- **Under 1,000 plain pages a month** → free plan. No card, 5 threads.
- **Around 10,000 rendered pages or 100,000 static pages** → Hobby, 20 threads.
- **SERP scraping at any real volume** → Startup at minimum. At 25 credits per Google or Bing request, a million credits is 40,000 searches.
- **Amazon or Walmart catalogues** → Business. At 5 credits per request, 3 million credits is 600,000 product pages, and Business is where country-level geotargeting starts.
- **You need to burst past the cap without changing tiers** → Scaling or above, for pay-as-you-go.
- **Above 500 concurrent threads** → Enterprise.

Two things to confirm before you pay: whether the 10% annual discount is worth locking a credit allowance you might outgrow, given that nothing rolls over, and that you're inside the 7-day refund window if the target list turns out harder than the trial suggested.
👉 [Compare all ScraperAPI plans and pick your tier](https://www.scraperapi.com/pricing?fp_ref=coupons)
