# directory-scraper

Turn public business directories into a clean, deduplicated leads
spreadsheet. Built as a reusable pipeline rather than a one-off script:
new sources plug in as adapters, and simple HTML directories need no code
at all — just a YAML file describing the page.

```
source adapter  ->  normalise  ->  dedupe/merge  ->  CSV + XLSX
```

## Why it exists

Scraping a listing page is the easy part. What takes the time is
everything after: phone numbers in six different formats, the same shop
listed twice with a trailing space, tracking junk glued onto every URL,
half-empty rows nobody can use. This repo is mostly that second part.

## Install

```bash
git clone https://github.com/USERNAME/directory-scraper.git
cd directory-scraper
pip install -r requirements.txt
```

Python 3.10+.

## Quick start

The bundled demo source is OpenStreetMap via the Overpass API — open
data, free, and it carries real contact fields, so you can see actual
output straight after cloning.

```bash
python -m directory_scraper \
  --adapter overpass \
  --location "Austin, TX" \
  --category cafe \
  --radius 8000 \
  --limit 300 \
  --out output/austin-cafes
```

Writes `output/austin-cafes.csv` and `output/austin-cafes.xlsx` (frozen
header, autofilter, columns sized to content).

Category accepts a friendly name (`cafe`, `dentist`, `plumber`,
`estate agent`) or any raw OSM tag:

```bash
--category "shop=florist"
--category "office=insurance"
```

> **One setup step:** OpenStreetMap's services reject anonymous clients.
> Put your own project URL and email in the User-Agent, or Nominatim
> returns 403:
>
> ```bash
> --user-agent "my-leads-tool/1.0 (+https://github.com/me/repo; me@example.com)"
> ```

## Scraping an HTML directory

Rather than a new scraper per site, describe the site once in YAML:

```yaml
start_url: "https://example-directory.com/{category}/{location}?page={page}"
item_selector: "div.business-card"

pagination: { start: 1, step: 1, max_pages: 20 }

fields:
  name: "h2.business-name"
  street: "span.street-address"
  city: "span.locality"
  rating: "span.rating-value"
  phone:
    selector: "a[href^='tel:']"
    attr: href
    strip_prefix: "tel:"
```

```bash
python -m directory_scraper --adapter html \
  --config configs/example_directory.yml \
  --location "austin-tx" --category "coffee-shops" --limit 200
```

Optionally follow each card through to the business's own page
(`detail_link` + `detail_fields`) to pick up emails the listing page
hides. Full annotated schema in [`configs/example_directory.yml`](configs/example_directory.yml).

## Output columns

`name · category · street · city · state · postcode · country · phone ·
email · website · rating · review_count · latitude · longitude · source ·
source_url`

Plus any `extra_fields` the config defines. Every row keeps its
`source_url`, so a buyer can verify any record by hand.

## What the cleaning layer does

| Problem | Handling |
|---|---|
| `(512) 555-0134`, `512.555.0134`, `1-512-555-0134` | normalised to `+15125550134` (`--country-code` sets the default) |
| `joescafe.com/?utm_source=directory` | `https://joescafe.com` — scheme added, tracking params dropped |
| `312 reviews` | `312` |
| Same business listed twice | merged, not dropped — blanks on one record filled from the other |
| Rows with no business name | discarded |

Duplicate detection matches on phone, on domain+name, and on name+street,
so a record missing one field is still caught by another.

## Being a good citizen

Every request goes through `PoliteSession`, which:

- reads and obeys `robots.txt`, including `Crawl-delay`
- rate limits with jitter (default ~1.5s between requests)
- retries `429`/`5xx` with exponential backoff and honours `Retry-After`
- sends an identifying User-Agent

`--ignore-robots` exists for sites you own or are licensed to crawl.
Collecting personal data, or crawling a site whose terms forbid it, is on
you — check before you point this at anything.

## Adding a source

```python
from directory_scraper.adapters.base import Adapter
from directory_scraper.models import Business

class MyDirectoryAdapter(Adapter):
    name = "mydirectory"

    def fetch(self):
        resp = self.session.get(f"https://example.com/{self.spec.location}")
        for row in parse(resp.text):
            yield Business(name=row["title"], phone=row["tel"])
```

Register it in `directory_scraper/adapters/__init__.py` and it appears as
a `--adapter` choice. Adapters stay dumb: yield what the page said, and
let the pipeline clean it.

## Tests

```bash
pip install pytest
python -m pytest tests -q
```

## Project layout

```
directory_scraper/
├── http.py              robots.txt, rate limiting, retries
├── models.py            the Business record
├── normalize.py         cleaning, merging, dedupe
├── export.py            CSV + formatted XLSX
├── cli.py               argparse entry point
└── adapters/
    ├── base.py          Adapter / SearchSpec interface
    ├── overpass.py      OpenStreetMap (runnable demo)
    └── html_directory.py  YAML-configured HTML directories
```

## Licence

MIT. OpenStreetMap data retrieved through the Overpass adapter is
© OpenStreetMap contributors, licensed under the ODbL — attribute it if
you redistribute it.
