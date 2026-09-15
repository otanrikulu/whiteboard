# STATS 401 Whiteboard — Industry Performance Across Market Conditions

An experimental **scaffold** of the full final project: a reproducible Python
data pipeline feeding five linked D3 visualizations. Not final — a sandbox to
experiment with design and analysis decisions.

## Structure

```
stats401-whiteboard/
├── index.html            dashboard shell (story-sequenced, 5 views)
├── css/whiteboard.css    project styles (layered on reused ../css/style.css)
├── js/
│   ├── config.js         front-end params (mirrors docs/data-spec.md)
│   ├── main.js           loads data, wires shared state, dispatches views
│   ├── viz1_timeline.js  S&P + VIX + condition ribbon
│   ├── viz2_scatter.js   return vs VIX, quadrants + typical band
│   ├── viz3_heatmap.js   industry × condition (avg / risk-adjusted toggle)
│   ├── viz4_network.js   per-condition correlation network (threshold slider)
│   └── viz5_comparison.js cumulative growth of $1 per industry
└── data/
    ├── build_data.py     acquire → clean → merge → classify → aggregate
    ├── raw/              cached source downloads
    ├── daily_panel.csv   date, sp500, sp_return, vix, trend, condition
    ├── industry_daily.csv long: date, condition, industry, ret
    ├── industry_by_condition.json  per industry × condition stats
    ├── correlations.json per-condition + overall 12×12 matrices
    └── meta.json         run params + counts (drives the Data panel)
```

## Run the pipeline

```
../.venv/Scripts/python data/build_data.py            # uses cached raw files
../.venv/Scripts/python data/build_data.py --refresh  # re-download sources
```

All tunable decisions live in the `CONFIG` block at the top of `build_data.py`.

## View the dashboard

Browsers block `file://` fetches, so serve over HTTP:

```
python -m http.server 8000
# then open http://localhost:8000/index.html
```

## Data sources

- S&P 500 — FRED (`SP500`)
- VIX — FRED (`VIXCLS`)
- 12 Industry Portfolios (Daily), **value-weighted** — Kenneth R. French Data
  Library. © Eugene F. Fama and Kenneth R. French.

See `../STATS401-Final-Group-Project/docs/data-spec.md` for locked definitions.
