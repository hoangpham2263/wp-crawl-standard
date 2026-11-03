# WP Crawl Standard: a standard for building crawl → WordPress / WooCommerce tools

A shared standard for building tools that **crawl data and import it into WordPress / WooCommerce** (posts, products, quizzes and more), distilled from many tools built in practice. When a new tool is needed: **give this file to an AI or developer, describe the data source and target, then build it within the framework below.**

---

## Mandatory rules (summary)

1. **One file per step:** `fN_<task>.py` with a `run() -> bool` function; `main.py` is only a menu that calls the steps.
2. **Each step reads the previous step's output from `data/`** and never calls its code directly.
3. **Re-running never breaks anything or creates duplicates:** skip finished items; prevent duplicates using `slug` or `sku`.
4. **Configuration lives in `config.json`** (source, selectors, WordPress, crawl speed), never hard-coded.
5. **Never commit passwords or data:** `config.json`, `data/` and CSV/Excel files go in `.gitignore`; `config.example.json` keeps passwords empty.
6. **Test first:** `test_limit = 5`, import posts as `draft`.
7. **Crawl politely:** wait 1–3 seconds between requests, 3 workers at most, with retries, and stop when blocked.
8. **Errors do not stop the pipeline:** failed items are logged to `data/logs/` and re-run separately later.
9. **Crawl only data you have the right to use;** if the source has an API or feed, use it.
10. **Every tool has a README:** source, target, how to run, data rights notes.

## Building a new tool: step by step

| Step | What to do | Expected result |
|---|---|---|
| **1. Survey the source** | Open the source site: is there a sitemap? An API in the Network tab? Are pages rendered with JS? Check `robots.txt` and data rights | Know where to get the URL list, and whether to use `requests` or Playwright |
| **2. Define the target** | Is the target a post, custom post type or WooCommerce product? Which categories, tags and ACF/meta are needed? | List of fields to capture (JSON schema in section 7) |
| **3. Create the skeleton** | Create the folder per section 2: `main.py`, `start`, `common.py`, `config.example.json`, `.gitignore`, `requirements.txt`, `README.md` | `./start` shows the menu |
| **4. Write f1: URL list** | Read the sitemap, txt file or CSV; filter with `url_include` / `url_exclude` | `data/f1_urls.json` |
| **5. Write f2: download HTML** | Download each URL per section 6, skipping existing files | `data/f2_html/*.html` |
| **6. Find selectors** | Open 2–3 sample pages and use Inspect to find selectors for title, content, images, category and date → add them to `config.json` | `selectors` section in the config |
| **7. Write f3: extract data** | Read HTML, extract by selector, clean and normalize | `data/f3_extracted/*.json`; open a few files to check by eye |
| **8. Write f4: download images** | Download the featured image and content images, named by slug | `data/f4_images/` + url → file map |
| **9. Prepare WordPress** | Create an Application Password; if ACF, meta or custom endpoints are needed, write `WORDPRESS_UPDATE.md` + the PHP snippet to add to the site | REST API sign-in works |
| **10. Write f5: import data** | Per section 9: check for duplicates, upload images, replace image URLs, create categories, create posts | `data/f5_import_state.json` |
| **11. Test run** | `test_limit = 5`, status `draft` → check the 5 posts in WordPress (content, images, categories, custom fields) | 5 correct drafts |
| **12. Full run** | `test_limit = 0`; watch the log; re-run failed items | All data imported |
| **13. Handover / archive** | Update the README, remove sensitive data before pushing to Git, revoke the Application Password if no longer needed | Clean repo with instructions |

---

## 1. Core idea: a multi-step pipeline, one file per step

```
f1 Get URL list  →  f2 Download HTML  →  f3 Extract data  →  f4 Download images  →  f5 Import to WordPress
   (sitemap/txt/csv)   (save .html files)   (save .json)        (save image files)     (REST API, with resume)
```

- Each step **reads the previous step's output from disk** and **writes its own output to disk**, so re-running one step does not repeat earlier steps, and data is easy to inspect along the way.
- Skip steps you do not need (for example, when the source is a Haravan CSV there is no f1/f2, so start from CSV extraction).

## 2. Folder structure

```
<source-name>-<type>-crawl/     # e.g. example-post-crawl, shop-product-crawl
├── main.py                 # step menu, "0 = run all"
├── start                   # #!/bin/bash → python3 main.py (chmod +x)
├── config.example.json     # sample config, NO real passwords
├── config.json             # real config, in .gitignore
├── .gitignore
├── requirements.txt
├── README.md               # source, target, how to run, data rights notes
├── common.py               # load_config, HTTP session, logger, shared utilities
├── f1_<step_name>.py       # each step has a run() -> bool function
├── f2_crawl_html.py
├── f3_extract_data.py
├── f4_download_images.py
├── f5_import_posts.py      # or f5_import_products.py
└── data/                   # all output, in .gitignore
    ├── f1_urls.json
    ├── f2_html/            # 1 .html file per URL
    ├── f3_extracted/       # 1 .json file per item
    ├── f4_images/
    ├── f5_import_state.json
    └── logs/
```

**Naming rules:** step files are `fN_<task>.py`; output of step N always has the `fN_` prefix in `data/`. Tool folder name: `<source>-<post|product|quiz>-crawl`.

## 3. Step contract

Each `fN_*.py` file has exactly one public function:

```python
def run() -> bool:
    """Return True when finished, False on a critical error (main stops the pipeline)."""
```

- Steps must be **safely re-runnable** (idempotent): skip items that already have output and only do what is missing.
- Print brief progress: `[f2] 120/540 ok, 3 errors`.
- Failed items are written to `data/logs/fN_errors.json` (URL + reason) for a separate re-run.

## 4. main.py (standard skeleton)

```python
#!/usr/bin/env python3
"""<Tool name>: menu for running each step of the crawl pipeline."""
from f1_urls import run as f1
from f2_crawl_html import run as f2
from f3_extract_data import run as f3
from f4_download_images import run as f4
from f5_import_posts import run as f5

STEPS = [("Get URL list", f1), ("Download HTML", f2), ("Extract data", f3),
         ("Download images", f4), ("Import to WordPress", f5)]

def main():
    while True:
        print("\n0. Run all")
        for i, (name, _) in enumerate(STEPS, 1):
            print(f"{i}. {name}")
        print("9. Exit")
        choice = input("Choose: ").strip()
        if choice == "9":
            break
        if choice == "0":
            for name, step in STEPS:
                print(f"\n--- {name} ---")
                if not step():
                    print(f"Stopped at step: {name}")
                    break
        elif choice.isdigit() and 1 <= int(choice) <= len(STEPS):
            STEPS[int(choice) - 1][1]()

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\nStopped.")
```

## 5. Configuration

`config.example.json` (committed), `config.json` (not committed):

```json
{
  "source": {
    "sitemap_url": "https://example.com/sitemap.xml",
    "url_include": ["/blog/"],
    "url_exclude": ["/tag/", "/page/"]
  },
  "selectors": {
    "title": "h1",
    "content": "article .entry-content",
    "image": "meta[property='og:image']",
    "category": ".breadcrumb a",
    "date": "time[datetime]"
  },
  "crawl": { "delay_min": 1, "delay_max": 3, "max_retries": 3, "workers": 3, "user_agent": "" },
  "wordpress": {
    "url": "https://your-site.com",
    "username": "",
    "app_password": "",
    "post_type": "posts",
    "status": "draft"
  },
  "test_limit": 5
}
```

- **Keep selectors in the config, not in code**, so changing the source only means editing the config.
- Allow reading the password from an environment variable to avoid writing it to a file: `WP_APP_PASSWORD` takes priority over `config.json`.
- `test_limit` > 0: each step processes only the first N items for a test run; 0 = run everything.
- Import posts as **draft** by default, to review before publishing.

## 6. Polite page downloads (f2)

- One shared `requests.Session`; a clear `User-Agent`; 20–30s timeout.
- Random wait of `delay_min`–`delay_max` seconds between requests; `workers` ≤ 3.
- Retry with backoff on network errors, 429 and 5xx; **stop completely** if blocked repeatedly.
- Detect encoding (chardet) before saving, and save as UTF-8.
- JavaScript-rendered pages: look for a JSON API in the Network tab first; use Playwright only when necessary.

## 7. Data extraction (f3)

- Each item produces one JSON file with a consistent schema:

```json
{
  "source_url": "",
  "slug": "",
  "title": "",
  "excerpt": "",
  "content_html": "",
  "categories": ["Parent", "Child"],
  "tags": [],
  "featured_image": "",
  "images": [],
  "published_at": "",
  "meta": {}
}
```

- Products also have: `sku`, `price`, `sale_price`, `stock`, `attributes`, `variations`.
- **Clean the content:** remove stray `script/style/iframe`, links back to the source site and needless inline attributes; keep basic HTML tags.
- Normalize: prices → integers, dates → ISO 8601, whitespace, slugs without diacritics.

## 8. Image download (f4)

- File names use the item slug + a sequence number; skip images already downloaded; record an `original url → file` map.
- Optionally compress or convert to WebP before upload to keep the media library light.

## 9. Import to WordPress (f5)

- Authenticate with an **Application Password** (Basic auth) through the REST API `/wp-json/wp/v2/...`; WooCommerce uses `/wp-json/wc/v3/products` with a consumer key and secret.
- **Prevent duplicates with a natural key**, not a sequence number: look up by `slug` (posts) or `sku` (products) before creating; if found, update or skip.
- **Resume with a finished list:** `data/f5_import_state.json` stores `{ "<slug>": <wp_id> }`, which is safer than storing "at item N".
- Upload images → replace image URLs in `content_html` with the new media URLs → create the post or product.
- Categories: cache `name → id`, and create the parent/child hierarchy if missing.
- Custom fields (ACF, post meta, custom endpoints): describe them in `WORDPRESS_UPDATE.md` with the PHP snippet that registers the meta or route on the site.
- Limit concurrency (semaphore of about 3) to avoid overloading the host.

## 10. Required .gitignore

```
config.json
.env
data/
venv/
__pycache__/
*.pyc
*.csv
*.xlsx
*.zip
.DS_Store
```

## 11. Security and data rights (read before crawling)

- **Never** commit `config.json`, application passwords or consumer keys; `config.example.json` holds empty strings only. If a password is committed by accident, revoke that Application Password in WordPress immediately.
- Crawl only data you **have the right to use**: a client's own old site, a shop owner's exported CSV data, licensed sources. If the source has an API or feed, use it.
- Respect `robots.txt` and the source site's terms; do not crawl personal data.
- Public repos: include only **the tool skeleton + sample config**, with no client data and no client names without permission.

## 12. README for each tool (short template)

```md
# <Tool name>
Source: <site/CSV>  →  Target: <WordPress post | WooCommerce product>

## Run
pip install -r requirements.txt
cp config.example.json config.json   # fill in URL, user, app password, selectors
./start                               # choose 0 to run all, or run each step

## Steps
f1 … f5 (what each step does, where its output goes)

## Notes
Use only with data you have the right to use.
```

## 13. New tool checklist

- [ ] Correct folder structure and `fN_*.py` file names; each step has `run() -> bool`
- [ ] Selectors and source live in the config; `config.example.json` has no passwords
- [ ] Complete `.gitignore` (config.json, data/, venv/ and so on)
- [ ] `test_limit` run on 5 items, imported as draft
- [ ] Re-runs create no duplicates (slug/SKU check), with resume through a state file
- [ ] Failed items logged separately, with a way to re-run them
- [ ] README states source, target, how to run and data rights notes

---

### Sample prompt for a new tool

> Read `CRAWL-STANDARD.md`. Build a `<name>-crawl` tool that follows the standard: the source is `<sitemap / CSV file / URL list>`, capture the fields `<…>`, and import into `<WordPress post type / WooCommerce>` with the category `<…>`. Selectors: `<…>`. Do a test run with `test_limit = 5` first.
