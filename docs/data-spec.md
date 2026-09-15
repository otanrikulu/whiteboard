# Data Specification — Single Source of Truth

_Project: Exploring Industry Performance Across Market Conditions_
_Last updated: 2026-09-15 · Status: living document_

This note pins down the data window, acquisition, cleaning rules, return
definition, and market-condition classification so every visualization is built
from one consistent set of decisions. **If a number or rule is used in code, it
must trace back to this file.**

### Adjustability principle (locked)

Every definition here is **provisional and revisable**. All thresholds and
choices (VIX cut, return sign rule, window, units, VW/EW, band edges) must be
implemented as **named parameters / config values in one place** — never as
hard-coded "magic numbers" scattered through the code. Changing a definition
later must mean editing this file + one config value and re-running, not hunting
through visualizations. This keeps the whole pipeline re-runnable if we revise
any decision.

---

## 1. Study window

| Item | Value |
|---|---|
| Start | **2016-09-15** |
| End — raw S&P/VIX | 2026-09-14 |
| **End — integrated panel** | **2026-07-31** |
| Length | ~10 years |
| Start governed by | S&P 500 (`SP500`) series availability on FRED (rolling 10-yr window) |
| End governed by | **12 Industry Portfolios** (French file built from 202607 CRSP DB → data ends 2026-07-31; the lagging series governs the panel end) |

VIX and the industry data are **trimmed** to this window. Because the industry
series is the lagging one, the **integrated (merged) panel ends 2026-07-31**,
even though S&P/VIX individually extend to 2026-09-14.

---

## 2. Sources & acquisition

All series acquired programmatically (API, not manual download) for reproducibility.

| # | Series | Source | Endpoint (locked) |
|---|---|---|---|
| 1 | S&P 500 | FRED | `https://fred.stlouisfed.org/graph/fredgraph.csv?id=SP500` |
| 2 | VIX | FRED | `https://fred.stlouisfed.org/graph/fredgraph.csv?id=VIXCLS` |
| 3 | 12 Industry Portfolios [Daily] — **Value-Weighted** table | Kenneth R. French Data Library | `https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/ftp/12_Industry_Portfolios_Daily_CSV.zip` |

- **Acquisition path (pipeline):** FRED CSV endpoints + French zip above — **no API key required**, all HTTP 200 confirmed.
- **Documented alternative (cite in report):** official FRED API
  `https://api.stlouisfed.org/fred/series/observations?series_id=SP500&api_key=KEY&file_type=json`
  (requires a free registered key; the public demo key returns HTTP 400).
- **Dataset 3 selection (locked):** the **Value-Weighted** daily table (the first
  of the two stacked tables in the file, titled *"Average Value Weighted Returns
  -- Daily"*). French pre-computes the market-cap weighting internally from CRSP;
  we do **not** source or compute any weights ourselves — we only select the VW
  series. Equal-Weighted table is not used.

### Attribution (required)

The industry return data **must be cited** as the Kenneth R. French Data Library.
The market-cap weighting methodology is French's, not ours. Suggested citation:

> Industry returns: Kenneth R. French Data Library, "12 Industry Portfolios
> (Daily)," value-weighted returns. https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/data_library.html
> © Eugene F. Fama and Kenneth R. French.

---

## 3. Cleaning rules

1. **Missing values** in FRED CSVs are encoded as **empty fields** (not `.`) on
   NYSE market holidays. Drop any row with a blank value.
2. **Trim VIX** from its full history (back to 1990) to the study window.
3. **Merge** S&P 500 and VIX with an **inner join on date**.
4. Holiday blanks differ slightly: S&P has 96 blanks in-window, VIX has 63, and
   every VIX blank is also an S&P blank (VIX blanks ⊂ S&P blanks). 33 dates are
   blank in S&P but carry a VIX value (post-2022 holidays). Requiring **both**
   values present drops the **union = 96 blanks**.
5. **Industry file (French):** parse the **Value-Weighted** table only (skip the
   prose header, the Equal-Weighted table, and the copyright footer). Dates are
   `YYYYMMDD` integers (convert to ISO). Returns already in percent. Missing =
   `-99.99` / `-999` → **0 in our window**. History runs back to 1926 → trim.
6. **Three-way merge (inner join on date):** S&P ∩ VIX ∩ Industries. Industry
   coverage ends 2026-07-31, so the integrated panel ends there.
7. Result: **2,482 aligned trading days** in the integrated panel.

_Verified during inspection:_
- S&P and VIX share the identical in-window date set (2,608 rows each; 0 dates
  unique to either series). S&P alone (dropping its 96 blanks) → ~2,512 days.
- French Value-Weighted dates align **perfectly** with the S&P trading calendar
  in the overlap (2,482 vs 2,482; zero mismatches).

---

## 4. Return definition & units

- **Daily simple return (S&P 500):** `r_t = (P_t / P_{t-1}) − 1` on the close.
  Chosen for interpretability with a general audience.
- _(Alternative to note in report: log returns `ln(P_t / P_{t-1})`.)_

### Units — **percent** everywhere (locked)

All returns are expressed in **percent**, to match French's industry table (which
is already in percent, e.g. `0.60` = 0.60%). The computed S&P return is therefore
scaled ×100: `r_t% = ((P_t / P_{t-1}) − 1) × 100`. Percent chosen over fractions
for axis/heatmap readability for a general audience.

---

## 5. Market-condition classification — 2×2 (four conditions)

Two independent cuts produce the proposal's four conditions:

| Axis | Rule | Boundary |
|---|---|---|
| Volatility | Calm if `VIX ≤ median`, Volatile if `VIX > median` | **median = 16.92** (single cut) |
| Trend | Up if `return ≥ 0`, Down if `return < 0` | **0** |

**Four condition labels:** `Calm-Up`, `Calm-Down`, `Volatile-Up`, `Volatile-Down`.

> Rationale: median split gives a balanced ~50/50 Calm/Volatile population and a
> defensible, data-driven boundary (no arbitrary "VIX = 20" cutoff).

---

## 6. VIX reference statistics (in-window, locked)

Computed on 2,545 non-blank VIX days within the study window.

| Stat | Value |
|---|---|
| min | 9.14 |
| p33 | 14.91 |
| **median** | **16.92** |
| p67 | 19.58 |
| p90 | 27.35 |
| mean | 18.62 |
| max | 82.69 (COVID crash, Mar 2020) |

---

## 7. Scatterplot design (Returns vs VIX)

- **Encoding:** x = daily S&P return, y = VIX, dot color = one of the four conditions.
- **Classification guides:** horizontal line at **VIX median ≈ 17** + vertical line at **return = 0** → four tinted quadrants.
- **Reference band:** light background band at **VIX 15–20** (p33→p67, the middle
  third) labeled **"typical VIX range."**
  - Purely **educational context** for non-finance users — it does **not** drive
    classification.
  - Label is **"typical," never "calm."** The true Calm/Volatile boundary is the
    median line only. (VIX ~25+ is ≥p85 = genuinely elevated, so it is not
    included in the "typical" band.)

---

## 8. Open decisions (not yet locked)

- Simple vs log returns (§4 default = simple).
- Whether the **Up/Down** axis uses the single-day return or a **trailing window**
  (e.g., 21-day) to capture *sustained* trend rather than one-day noise.
- Risk-adjustment (Sharpe-like `mean/vol`) for the industry heatmap.

### Locked since first draft

- Dataset 3 = 12 Industry Portfolios [Daily], **Value-Weighted** table (§2).
- Units = **percent** everywhere (§4).
- Integrated panel end date = **2026-07-31**, 2,482 trading days (§1, §3).
- French attribution required (§2).
