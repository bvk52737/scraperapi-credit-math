# web scraping golang: When Colly and chromedp Hit a Wall, Plus What ScraperAPI Actually Costs in Credits

Most "web scraping golang" tutorials stop at the same point: a working goquery or Colly example against a practice site like books.toscrape.com. That's fine until you point your scraper at a real target with Cloudflare in front of it, and every request comes back 403 no matter what User-Agent you set. This article covers the standard Go scraping stack, the exact points where it breaks, and how a scraping API like ScraperAPI plugs into a Go project — including the credit math most pricing pages don't spell out, so you can figure out your real monthly cost before you commit.

> **Quick take:** if your Go scraper already works but gets blocked, ScraperAPI handles proxies, CAPTCHAs and JS rendering for you — the Hobby plan is $49/mo (or $44.10/mo billed annually) with 100,000 credits, and there's a 7-day trial with 5,000 credits that needs no credit card. 👉 [Check ScraperAPI's current plans and prices](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## The standard Go scraping stack — and its three breaking points

Go is genuinely good at scraping. Goroutines let you fetch hundreds of pages concurrently with low memory overhead, the standard library ships a capable HTTP client, and the compiled binary deploys anywhere with no runtime dependencies. Most tutorials converge on the same three tools, each with a specific job:

- **net/http + goquery** for static pages. goquery gives you jQuery-style selectors (`Find()`, `Each()`, `Attr()`, `Text()`), and Go's type system catches extraction errors at compile time.
- **Colly** for structured crawling. It's the most popular Go scraping framework, with request scheduling, rate limiting, caching, concurrency, automatic cookie handling and robots.txt compliance built in.
- **chromedp** for JavaScript-rendered pages, since it drives a real Chrome.

The tutorials rarely dwell on where this stack fails. Three places, in practice:

**1. The default User-Agent gives you away instantly.** Go's net/http client sends `Go-http-client/2.0` as its UA. Many sites return 403 without a browser-looking header, so you have to set a Chrome UA plus `Accept` and `Accept-Language` manually on every request.

**2. TLS fingerprints are the deeper problem.** Even with a perfect Chrome UA, net/http produces a non-browser TLS fingerprint. Anti-bot systems like Cloudflare, DataDome and Akamai check JA3/JA4 fingerprints, and they'll flag your Go client regardless of headers. Fixing this means a real browser or a service that has already solved it.

**3. chromedp is heavy to run at scale.** It works for JS-rendered pages, but you're maintaining Chrome instances, and a CAPTCHA still stops it cold.

One more gotcha that deserves more attention than it gets: Go's `http.Client` has no default timeout. A hung connection will block a goroutine indefinitely. Always set something like `Timeout: 30 * time.Second`, or use `context.WithTimeout`.

## Where a scraping API fits into a Go project

A scraping API moves the blocking problem server-side: instead of rotating proxies and solving CAPTCHAs in your Go code, you send one request to the API endpoint and get back clean HTML. ScraperAPI is one of these services — proxies, CAPTCHA handling and headless browsers are its job, with a 40M+ IP pool across 50+ countries.

The integration for Go is a plain HTTP GET. That's the whole thing:

go
scraperURL := fmt.Sprintf("https://api.scraperapi.com?api_key=%s&url=%s&render=true", apiKey, targetURL)


Add `render=true` for JavaScript-heavy pages. The service renders the page and returns HTML you can feed into goquery or chromedp for parsing. Two more parameters matter in real projects:

- `session_number=1234` keeps the same session across requests, useful for multi-step flows like logins or pagination that tracks state.
- `country_code=us` pins requests to IPs from a specific country.

ScraperAPI's own blog documents a chromedp integration path worth knowing about, because it's counterintuitive: the recommended approach is to let ScraperAPI render the page (`render=true`) and use chromedp purely as a parser on the returned HTML. The method people assume — plugging ScraperAPI into chromedp as a Chrome proxy — is the one their guide explicitly warns against. They report hitting `net::ERR_INVALID_ARGUMENT` and `net::ERR_NO_SUPPORTED_PROXIES`, with TLS and authentication often failing silently in headless Chrome. So in this setup, rendering and parsing swap roles from what most people expect.

There's also a third-party Go SDK on GitHub (`gopkg.in/scraperapi.v1`, MIT licensed) that wraps the endpoint with typed options like `WithRenderJS()`, `WithSessionNumber()`, `WithCountryCode()` and `WithAutoParse()`, plus an `Account()` call to check your quota. It's community-maintained, so the raw endpoint is the safer long-term dependency.

Either way, the same Go hygiene applies: load the API key from a `.env` file instead of hardcoding it, wrap requests in a 30-second timeout, and retry failed requests up to 3 times with a 2-second backoff.

You can test the whole thing against the free tier before paying anything: 👉 [Sign up for the 7-day trial — 5,000 API credits, no credit card](https://www.scraperapi.com/signup?fp_ref=coupons)

## Full plan comparison

ScraperAPI currently lists seven tiers. All of them include JS rendering, premium residential and mobile IPs, CAPTCHA and anti-bot handling, automatic retries, unlimited bandwidth and a 99.9% uptime guarantee — the tiers differ mainly in credits, concurrency and geotargeting.

| Plan | Monthly price | Annual (per month) | API credits | Threads | Get it |
| --- | --- | --- | --- | --- | --- |
| Hobby | $49/mo | $44.10/mo | 100,000 | 20 | [Start with Hobby](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | $149/mo | $134.10/mo | 1,000,000 | 50 | [View Startup pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | $299/mo | $269.10/mo | 3,000,000 | 100 | [View Business pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling (most popular) | $475/mo | $427.50/mo | 5,000,000 | 200 | [View Scaling pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | $975/mo | $877.50/mo | 10,500,000 | 300 | [View Professional pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | $1,975/mo | $1,777.50/mo | 21,500,000 | 500 | [View Advanced pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | Custom | Custom | 22M+ | 500+ | [Contact sales for Enterprise](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

There's also a permanent free plan: 1,000 API credits with a maximum of 5 concurrent connections, separate from the 7-day trial.

## The credit math most reviews skip

ScraperAPI charges per credit, and a "standard page" costs 1 credit. That's the number every summary quotes. What actually determines your bill is the site-specific surcharge list, which the tutorials almost never include:

- Standard page: **1 credit**
- Amazon product page: **5 credits**
- Google or Bing search (including subdomains): **25 credits**
- LinkedIn: **30 credits**
- Sites behind Cloudflare, Datadome or PerimeterX: **+10 credits per request** when ScraperAPI bypasses the protection

This changes what a plan is worth. Hobby's 100,000 credits is 100,000 plain pages — but only about 20,000 Amazon pages, or 4,000 Google searches. If your Go scraper's job is SERP monitoring, Hobby runs out after roughly 130 Google queries per day; if it's scraping regular e-commerce catalogs, it's comfortable.

Two dashboard features help you keep this under control. The Domain Cost Estimator shows the exact credit cost for any URL before you hammer it, and a `max_cost` parameter caps what a single request can consume — handy when you don't know whether a target sits behind a protection layer that adds 10 credits a hit. Failed requests are refunded, so retries don't silently drain your balance.

## What a year of ScraperAPI actually costs

Annual billing takes 10% off every tier. In dollars, that's $58.80/year saved on Hobby ($44.10 vs $49 per month), $178.80 on Startup, and $570 on Scaling. Nothing exotic, but it's the difference between "roughly $600" and "roughly $530" for a year of Hobby.

The pricier surprise comes from usage overruns, and the policy differs by tier in ways the pricing table doesn't surface:

- When credits run out, scraping pauses by default until the next cycle.
- **Auto-upgrade** (opt-in, available on Hobby through Professional) bumps you to the next plan automatically when you run dry — at that plan's higher price.
- **Pay-as-you-go** (only on Scaling and above) keeps scraping past your limit at a fixed per-credit rate, capped at a monthly spend ceiling you set yourself.

Concurrency overruns are gentler: exceeding your thread limit returns a 429 and the request is refused rather than queued, with no extra charge.

One restriction that catches people: **Hobby and Startup only support US and EU geotargeting.** Country-level targeting anywhere in the world starts at Business ($299/mo). If your Go scraper needs, say, Australian or Japanese IPs, the $49 plan won't do it regardless of credits.

## Which plan fits which Go project

Scraping static product pages with a Colly crawler that hits a few thousand pages a month → Hobby is enough, and the annual rate makes it $44.10/mo.

Google SERP data, Amazon pricing, or anything behind Cloudflare → do the credit math first with the surcharges above, then pick between Startup and Business. A SERP tracker doing 20,000 Google queries a month needs 500,000 credits just for searches.

Traffic that spikes unpredictably, or a production pipeline where stopping is worse than overspending → Scaling is the entry point for pay-as-you-go, which is presumably why it's the "most popular" tier among teams rather than individuals.

Just experimenting with web scraping in Go → don't pay anything yet. The free plan's 1,000 credits plus the 5,000-credit trial will tell you whether your target site actually needs a scraping API. If your targets are static pages with no bot protection, plain goquery with proper headers and timeouts does the job for $0.

## FAQ

**Do I need to change much of my Go code to use ScraperAPI?** No. Since it's a REST endpoint, you swap your target URL for the ScraperAPI URL and keep the rest — goquery parsing, goroutine concurrency, retry logic — untouched.

**Does ScraperAPI have an official Go SDK?** Not one we could verify. The Go SDK on GitHub is third-party and MIT-licensed; the official documentation treats the HTTP endpoint as the primary integration, which is also what their chromedp guide uses.

**What happens if a request fails?** Credits for failed requests are refunded, and all plans include automatic retries.

**Is there a refund or trial period?** There's a 7-day free trial with 5,000 API credits and no credit card required, which is the practical way to evaluate fit before any payment.

**Why do my credits disappear faster than my request count?** Site-specific credit costs. Check the Domain Cost Estimator in the dashboard for any URL you're targeting, and set `max_cost` on requests to expensive domains.

## Before you pick a plan

Figure out your credit burn first, because it's the only thing that really determines which tier you need: count your monthly requests, multiply by the per-site credit cost (1 for standard pages, 25 for Google, 5 for Amazon, +10 if your target sits behind Cloudflare or Datadome), and match that to the table above. If the number lands anywhere near 100,000, start on Hobby with annual billing and turn on auto-upgrade as your safety net; if you need non-US/EU IPs, skip straight to Business.

And if you haven't confirmed your target site actually blocks Go clients yet, test that before spending anything — 👉 [grab the free trial credits and run your real target through ScraperAPI once](https://www.scraperapi.com/signup?fp_ref=coupons). The Domain Cost Estimator will show you what each request costs, and you'll know within five minutes whether the service earns its keep for your specific scraper.
