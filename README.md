# Ford Canada VIN recall scraper

This project automates lookups on the Ford Canada recall support page for many vehicle identification numbers (VINs), extracts year, model, recalls, and customer satisfaction program (CSP) details, and writes the results to CSV.

The implementation lives in `main.py` and uses **Playwright** (async Chromium) because the target experience is driven almost entirely by JavaScript and dynamic content.

---

## Requirements

- Python 3.10+ (recommended)
- Playwright browsers installed after `pip install playwright` (typically `playwright install chromium`)

---

## Installation

From the project directory:

```bash
pip install playwright openpyxl
playwright install chromium
```

Use `openpyxl` only if your input file is `.xlsx`. CSV input uses the standard library.

---

## Configuration (`main.py`)

| Setting | Purpose |
|--------|---------|
| `INPUT_FILE` | Path to `.xlsx` or `.csv` containing a **`VIN`** column (header name matched case-insensitively). |
| `OUTPUT_FILE` | Path where rows are appended in CSV format. |
| `URL` | Ford CA recalls lookup URL (default: `https://www.ford.ca/support/recalls/`). |
| `TIMEOUT_MS` | Default timeout (ms) for navigation, waits, and clicks. |
| `MAX_TIMEOUT_RETRY_SESSIONS` | Maximum full **rounds** of retries for VINs that hit a Playwright timeout. |

Adjust these constants at the top of `main.py` before running.

---

## How to run

```bash
python main.py
```

Place your input workbook or CSV beside `main.py` (or update `INPUT_FILE` to its path).

---

## What the script does (process flow)

1. **Load VIN list** — Reads non-empty cells from the `VIN` column. Supports Excel (`.xlsx`) and comma-separated `.csv` with encoding fallbacks (UTF‑8 variants, Windows-1252, Latin-1) for messy exports.

2. **One Chromium instance per VIN** — For each VIN in order:
   - Launches a new browser context and page (no leftover state from the previous VIN).
   - Opens the Ford recalls URL.
   - Waits for the VIN input, enters the value, submits (Enter).

3. **Stabilization** — Optionally waits for `networkidle` (within timeout), then waits for vehicle year/model from `//div[@class="vehicle-information-ym"]`. The **first whitespace-separated token** is treated as year; everything after as model (handles multi-word model names).

4. **Recall / CSP panel** — Waits until the recalls/CSP shell is attached in the DOM (several anchors: `#recalls-content`, `#recalls-section`, `#csp-section`, relevant `h2` nodes, plus fallbacks). Uses heading text and CSS classes (`no-recalls`, `no-csp`) plus XPath for “No Recalls” / “No Customer Satisfaction Programs” to decide whether a side truly has accordion data.

5. **Expand accordions** — If recall or CSP has real data, every collapsed accordion under `#recalls-content` is clicked open before scraping (buttons with `aria-expanded="false"` inside `accordion-item` rows).

6. **Structured extraction**
   - **Recalls**: Per row under `#recalls-section`: title from `accordion-titles/div`, status from `accordion-titles/span`, campaign from the `id` attribute of `[data-testid="accordion-description"]`; description duplicates the title.
   - **CSP**: Per row under `#csp-section`: same title/campaign pattern; next steps via a sibling of the element whose text indicates “Next Steps”, scoped inside each accordion item.

7. **CSV write** — On success, rows are appended: **Recall** section rows first, then **CSP** rows. Column order is `VIN`, `Section`, `Year`, `Model`, `Title`, `Description`, `Campaign`, `Status`, `Next Steps`. Placeholder rows are written when there are genuinely no recalls or no CSP rows for that VIN.

8. **Teardown and hygiene** — After each VIN, **cookies are cleared** and the **browser disk cache is cleared** via Chromium DevTools (`Network.clearBrowserCache`), then the page, context, and browser close. This avoids leaking session or cached responses into the next VIN.

9. **Retries** — If a step raises Playwright `TimeoutError`, that VIN is **not** written to CSV in that attempt. Failed VINs are collected and run again in subsequent **rounds**, each round again using **one fresh browser per VIN**, until either all succeed or `MAX_TIMEOUT_RETRY_SESSIONS` is reached.

---

## Output CSV semantics

- **Successful VIN**: One or more Recall rows followed by one or more CSP rows (each program is its own CSP row).

- **No open recalls**: A single Recall row with `Status` set to `"No Recalls"` and empty title/campaign fields as appropriate.

- **No CSP programs**: A single CSP row with `Next Steps` set to `"No CSP"` where no programs exist.

If you ever change header names or column order in code, prefer starting a **new** output filename or deleting the old file so the CSV header stays consistent.

---

## Challenges encountered (project history)

The behavior of this scraper evolved through several iterations. Problems we hit and how we shaped the solution:

1. **Batch input and CSV encoding** — Early runs assumed a single hardcoded VIN. We switched to loading many VINs from files. Windows/Excel exports often use encodings outside UTF‑8 (`cp1252`, etc.), which caused `UnicodeDecodeError` until we added sequential encoding attempts for CSV reads.

2. **Excel-first workflow** — Input moved from CSV to `.xlsx` for operator convenience. That required `openpyxl` and a clear error if it is missing.

3. **Many combinations of Recall vs CSP UI** — The page can show: only recalls, only CSP, both, neither, or “No Recalls” / “No CSP” headings without accordions. Naive scraping mixed sections or relied on timeouts. We aligned logic with headings (`h2`), section classes, and separate branches so one side can be empty while the other still extracts.

4. **DOM differences and timing** — Production markup did not always expose `#recalls-section` / `#csp-section` when we first expected, or elements were only **attached** before paint. Waits were tightened to use **attachment** where visibility was flaky, and optional fallbacks under `#recalls-content` and `data-testid="recalls-csp-accordions"` were added so the script does not depend on a single attribute set.

5. **Extracting the right fields** — Initial parsing used generic label/value pairs inside accordions. After verifying the live DOM in DevTools, extraction was rewritten around **explicit XPaths** and structure: title and status per recall row, campaign as the `id` on `accordion-description`, CSP next steps as a following sibling of the “Next Steps” label, and year/model from a dedicated `vehicle-information-ym` node with “first token = year, rest = model”.

6. **Timeout rows polluting the CSV** — When a wait failed, swallowing errors still produced placeholder “No Recalls / No CSP” rows. Retry logic now **does not save** timed-out attempts and re-queues those VINs instead.

7. **Session isolation** — Even with correct selectors, reused cookies and cached assets sometimes led to unstable or inconsistent results across many lookups. The current design uses **one new browser lifecycle per VIN** and **`purge_cookies_and_cache`** (cookies + DevTools cache clear) before closing Chromium, then a completely new instance for the next VIN.

---

## Why Playwright instead of `requests` or Scrapy?

The Ford recall experience is implemented as a **single-page-style, JavaScript-heavy** application:

- Meaningful content—the vehicle block, recalls, and CSP—is not delivered as plain HTML on first response. Scripts fetch data asynchronously, hydrate the DOM, and drive accordions and state.

- Browser traffic often includes payloads that appear **opaque or encoded** in network logs; decoding and aligning them with rendered copy would require reversing minified bundles, handshake cookies, tokens, anti-bot behavior, or server-only decoding steps. Whatever is decoded for display happens in the combination of frontend logic and backend APIs—we do **not** reimplement that pipeline.

Libraries like **`requests`** or **Scrapy** excel when you own stable URLs with predictable HTML or public JSON endpoints. Here, the **surface we must match is the rendered page** after JS runs. **Playwright** drives a real browser: it executes the same scripts Ford ships, submits the same VIN interactions, waits for selectors that match what a user sees, and reads DOM text and attributes—which is reliable for this problem class despite being heavier than a bare HTTP client.

---

## Playwright pitfalls we mitigated

- **Session and cookies**: A long-lived tab can accumulate cookies, storage, or cache entries that subtly change routing, A/B splits, or rate limits across consecutive VIN lookups.

- **Mitigation**: After each completed (or failed) VIN scrape, **clear cookies** on the Playwright **`BrowserContext`** and clear the Chromium **disk cache** through CDP, then **close the browser**. The next VIN starts with no shared browsing state beyond what the next navigation loads.

Taken together: **automate the UI the way a careful human would**, use **explicit DOM contracts** (`h2`s, `#recalls-section`, `#csp-section`, `data-testid`, accordions), and **reset browser state between VINs** so intermittent extraction issues are less likely than with a reused session.

---

## License / use

This tool is intended for lawful, policy-compliant scraping of publicly available recall information for operational or research use. Respect Ford’s terms of service, robots guidance, rate limits, and applicable law.
