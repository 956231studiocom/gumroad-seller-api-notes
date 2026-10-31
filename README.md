# Gumroad Seller API — Field Notes (verified 2026-10-31)

Field-tested notes on what the Gumroad **API v2** can and cannot do when you are
trying to make a product show up in **Gumroad Discover**. Everything below was
verified against a live seller account, not copied from docs.

If you sell on Gumroad and your products get zero views, this is probably why.

---

## TL;DR

| Lever | API writable? | Notes |
|---|---|---|
| `taxonomy_id` (category) | **Yes** | Default is `266` = `other`, which is effectively invisible in Discover |
| `tags` | **Yes** (max 5) | Gumroad normalises/drops some phrases; put the money keyword first |
| `covers` (product images) | **Yes** via `POST /v2/products/:id/covers` | `url` param works; multipart file upload silently returns 0 covers |
| `discoverable` | **No** | `PUT` returns HTTP 200 but the value never changes |
| `thumbnail_url` | **No** (server-side) | Stays `null`; there is no API path to generate it |

The blocking fact: **Discover search results require a non-null `thumbnail_url`**,
and `thumbnail_url` is derived from `cover.thumbnail`, which the API leaves `null`.

---

## 1. Writing a cover

`POST /v2/products/:id/covers` accepts a `url` field:

```bash
curl -X POST "https://api.gumroad.com/v2/products/$PID/covers" \
  -d "access_token=$TOKEN" \
  -d "url=https://example.com/cover-1280x720.png"
```

Response contains a cover object:

```json
{
  "url": "https://public-files.gumroad.com/xxxx",
  "original_url": "https://public-files.gumroad.com/yyyy",
  "thumbnail": null,
  "id": "d51349c13ae56ac68e12d385b4de2eb6",
  "type": "image",
  "filetype": "png",
  "width": 670,
  "height": 376,
  "native_width": 1280,
  "native_height": 720
}
```

The image is stored and the product page picks it up (`og:image` starts working),
**but `thumbnail` stays `null`**.

`multipart/form-data` upload with a `file` part does **not** work — it returns
`{"success": true, "covers": [], "main_cover_id": null}`. Use the `url` form.

## 2. Writing a category

```bash
curl -X PUT "https://api.gumroad.com/v2/products/$PID" \
  -d "access_token=$TOKEN" -d "taxonomy_id=65"
```

Known valid IDs observed (they are **not** contiguous):

- `65` → `software-development`
- `66` → `programming`
- `69` → `web-development`
- `266` → `other` (the default — avoid it)

Invalid IDs are rejected on write; the product keeps its previous category.

## 3. Writing tags

Max 5 tags per product. Gumroad normalises them, so longer phrases are often
folded into shorter canonical tags:

```
"sales tax nexus, economic nexus, us sales tax"  →  ["sales tax", "nexus", "economic nexus"]
```

Put the highest-intent phrase first — order appears to affect which canonical
tags survive.

## 4. What does *not* work

- **`discoverable`** — `PUT` with `discoverable=true` returns HTTP 200 and the
  product object still reports `null`. It is not a public API field.
- **`thumbnail_url`** — read-only, server-generated. No API endpoint accepts it.
- **`main_cover_id`** — you can read it, you cannot set it directly; it is chosen
  from the `covers` array.

## 5. How Discover actually filters

Scraping `https://gumroad.com/discover?query=...` returns a React shell; the real
payload is embedded in a `data-page` attribute and parses as JSON:

```python
import re, html, json, urllib.request
h = urllib.request.urlopen("https://gumroad.com/discover?query=tax+nexus").read().decode()
payload = json.loads(html.unescape(re.search(r'data-page="(.*?)">', h, re.S).group(1)))
products = payload["props"]["search_results"]["products"]
```

Across a sample of 111 Discover results:

- **34 had zero ratings** → sales/reviews are *not* a hard requirement
- **111 / 111 had a non-null `thumbnail_url`** → the cover thumbnail *is* the gate

So: no thumbnail, no Discover. Confirmed on a live account with 10 published
products, correct categories, correct tags, and a valid cover — all still absent
from `?query=<own product name>`.

## 6. Practical workaround

If the API cannot produce a thumbnail, the only remaining path is the seller
dashboard UI (upload the cover there and let the front-end generate the
thumbnail). Budget for a manual step per product; there is no bulk endpoint.

---

## Why this repo exists

We build small tools for cross-border sellers and got tired of guessing which
Gumroad fields are actually writable. These notes are the field log.

Related, if useful:
- [US sales tax nexus tracker](https://956231studio.com/go/tax-nexus)
- [Chargeback reply templates](https://956231studio.com/go/chargeback)
- [DDP landed cost calculator](https://956231studio.com/go/ddp)

Corrections welcome — open an issue if any of the above behaves differently on
your account.

---

Maintained by [956231 Studio](https://956231studio.com/) — practical packs for sellers and engineers, built and tested in production: [browse all packs](https://956231studio.com/packs/).
