<p align="center"><a href="https://apify.com/DevilScrapes/aliexpress-products-scraper"><img src="assets/banner.svg" alt="AliExpress Products Scraper — Python example" width="100%"></a></p>

# AliExpress Scraper in Python: product prices, ratings and sold counts in pandas

<p align="center">
<a href="https://apify.com/DevilScrapes/aliexpress-products-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/aliexpress-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/aliexpress-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

Here is a compact Python client for the **[AliExpress Products Scraper](https://apify.com/DevilScrapes/aliexpress-products-scraper)**, an Actor that runs on Apify. You describe what to look up, as search keywords, item links or numeric product IDs, and the Actor brings back one record per AliExpress product with its price and currency, average star rating, the number behind the "sold" badge and its gallery images. The work of fetching AliExpress pages happens on Apify's side; this script just starts the run and writes `results.json` and `results.csv`. That makes it a natural first cell in a notebook or one step in a scheduled data job.

> **Actor:** [AliExpress Products Scraper](https://apify.com/DevilScrapes/aliexpress-products-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [aliexpress-scraper-javascript](https://github.com/devilscrapes/aliexpress-scraper-javascript)

## 🔥 What you get

- **Three ways in**: search keywords, `aliexpress.com/item/...` links and bare numeric product IDs can be mixed in a single run.
- **Each product appears once**: a `product_id` already written in the run is skipped, even when it turns up again through another keyword, a URL or the ID list.
- **Demand signal on search rows**: the "sold" badge from the results page is parsed into an integer `order_count`, alongside price, currency, star rating and images.
- **Plausibility checks on every row**: the price must be above zero and `product_url` must contain the row's own numeric ID, so a page that loaded without real product data never becomes a record.
- **Country-matched routing**: by default the Actor uses residential Apify Proxy in `shippingDestinationCountry` and, for the countries it has a currency for, skips a keyword whose first results page is priced in the wrong one.
- **Clear failure semantics**: a keyword with no matches finishes with zero rows and a status message; the run fails only when every one of its inputs went unresolved.

## 🎯 Use cases

### Validate a product idea with numbers

Search a few phrases for the item you are considering and look at the spread of prices next to the sold counts. Many listings with large `order_count` values at a similar price tell you the item sells, and roughly what the market will pay.

### Build a price history AliExpress does not show

Schedule the script with cron, keep each run's CSV under a dated name and concatenate them in pandas. Grouping by `product_id` over time gives you a per-item price series you can plot or test for drops before a sale event.

### Rank candidates for a dropshipping or Amazon catalog

Filter out items with weak ratings, then score the rest by demand against price inside each keyword. The pandas recipe further down does exactly this and writes a candidate list you can hand to whoever orders samples.

### Turn a supplier spreadsheet into current prices

If you already track a list of AliExpress item links, pass them as `productUrls` and refresh title, price, rating, review count and images in one run. Each row's `product_url` is canonical, so joining back to your sheet is straightforward.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/aliexpress-scraper-python.git
cd aliexpress-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [AliExpress Products Scraper](https://apify.com/DevilScrapes/aliexpress-products-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

If you just want the data in Python, this snippet is enough. `ApifyClient` launches the Actor, waits for the run to end and then yields each dataset item.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/aliexpress-products-scraper").call(run_input={'searchKeywords': ['wireless earbuds'], 'productUrls': [], 'productIds': [], 'maxResultsPerKeyword': 10, 'shippingDestinationCountry': 'US', 'proxyConfiguration': {'useApifyProxy': True}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` sends `input.json` to the Actor exactly as stored. The sample searches one phrase and caps it at ten products, with prices for a US shopper, which is a quick and inexpensive first run. Put item links in `productUrls` or IDs in `productIds` for direct lookups; those take longer, because each item page is rendered in a browser.

```json
{
  "searchKeywords": [
    "wireless earbuds"
  ],
  "productUrls": [],
  "productIds": [],
  "maxResultsPerKeyword": 10,
  "shippingDestinationCountry": "US",
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `searchKeywords` | array | no | `[]` | — |
| `productUrls` | array | no | `[]` | — |
| `productIds` | array | no | `[]` | — |
| `maxResultsPerKeyword` | integer | no | `20` | — |
| `shippingDestinationCountry` | string | no | `"US"` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

Every dataset item is a single AliExpress product. Coverage depends on how a row was found: keyword rows carry `order_count` and `search_keyword` but no `review_count`, while URL and ID rows carry `review_count` but no `order_count`. In the current version `variants` is an empty list and `seller_name`, `category` and `shipping_cost_estimate` are `null` on every row, so do not build logic on them yet.

| Field | Type | Always present |
|---|---|---|
| `product_id` | string | yes |
| `title` | string | yes |
| `price` | number | yes |
| `currency` | string | yes |
| `rating` | number or null | no |
| `review_count` | integer or null | no |
| `order_count` | integer or null | no |
| `variants` | array | yes |
| `shipping_cost_estimate` | number or null | no |
| `shipping_destination` | string | yes |
| `product_url` | string | yes |
| `image_urls` | array | yes |
| `seller_name` | string or null | no |
| `category` | string or null | no |
| `search_keyword` | string or null | no |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.05 |
| Product scraped | $0.002 |

Prices as configured on 2026-09-01. The [Actor page](https://apify.com/DevilScrapes/aliexpress-products-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

This pandas example (`pip install pandas`) prints a per-keyword overview, keeps products rated 4.5 or better that show a sold count, scores each by demand rank minus price rank within its keyword and writes the top 15 to `candidates.csv`.

```python
import json

import pandas as pd

items = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
proven = items.dropna(subset=["order_count", "rating"])
proven = proven[proven["rating"] >= 4.5].copy()

proven["price_rank"] = proven.groupby("search_keyword")["price"].rank(pct=True)
proven["demand_rank"] = proven.groupby("search_keyword")["order_count"].rank(pct=True)
proven["score"] = proven["demand_rank"] - proven["price_rank"]

summary = items.groupby("search_keyword").agg(
    products=("product_id", "nunique"),
    median_price=("price", "median"),
    with_sold_badge=("order_count", "count"),
)
print(summary.to_string())

picks = proven.sort_values("score", ascending=False).head(15)
picks[["search_keyword", "title", "price", "currency", "rating", "order_count", "product_url"]].to_csv(
    "candidates.csv", index=False
)
print(picks[["price", "rating", "order_count", "title"]].to_string(index=False))
```

## ❓ FAQ

### Is an AliExpress account or affiliate API key required?

No. The Actor works from the pages a signed-out shopper sees. An Apify API token in `APIFY_TOKEN` is the only secret this script reads.

### Why is `order_count` empty for some products?

It comes from the sold badge on search results, and not every listing shows one. Rows found through `productUrls` or `productIds` never have it, because the item page data the Actor reads does not include it. A missing badge is reported as `null`, never as zero.

### Which currency are prices in?

The one AliExpress shows for the destination country, recorded in `currency` on each row. Keep `shippingDestinationCountry` fixed between scheduled runs, or your price series will mix currencies.

### Can I use it from Jupyter or Airflow?

Yes. In Jupyter, import the module and pass the same arguments the command line takes to `main.main`, as a list. In Airflow, a BashOperator running `python main.py` is enough: the script returns exit code 1 when the Actor run fails, so the task goes red without extra checks.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/aliexpress-products-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [AliExpress Products Scraper on Apify Store](https://apify.com/DevilScrapes/aliexpress-products-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/aliexpress-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [eBay Product & Listings Scraper](https://apify.com/DevilScrapes/ebay-product-listings-scraper) — the same products' asking prices on eBay.
- [Mercado Libre Search Results Scraper](https://apify.com/DevilScrapes/mercadolibre-scraper) — marketplace search results for Latin American buyers.
- [Shopify Store Leads Scraper](https://apify.com/DevilScrapes/shopify-store-leads-scraper) — find the Shopify stores that resell such items.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/aliexpress-products-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with AliExpress or Alibaba Group.
