# Pack Oracle — MTG EV Analyzer

A full-stack analytics application for evaluating Magic: The Gathering booster packs and boxes through expected card value, simulated opening outcomes, and historical product trends.

**Python · Flask · NumPy · SciPy · SQLite · JavaScript · Auth0 · Docker**

> This repository is a public showcase of Pack Oracle, originally named MTG EV Analyzer. The application source is maintained privately. This repository contains project documentation and screenshots; it is not a runnable distribution.

## What it does

A booster product's average card value can look appealing even when most openings return less than that average. Pack Oracle pairs weighted expected-value calculations with simulated outcome distributions so collectors and sellers can see the typical opening, the spread, and the rare high-value tail.

The application follows three connected workflows:

1. **Browse:** search a catalog of packs and boxes, sort tracked metrics, and save products to a personal watchlist.
2. **Analyze:** select a booster configuration and market, inspect the biggest card hits, and simulate the tracked card value of one or more products.
3. **Compare:** examine historical price, opening ROI, and skewness trends for two to five products from a shared starting date.

Coverage depends on available booster configurations, prices, and saved metrics. Play, Draft, Set, and Collector boosters are supported where data exists; legacy MTGJSON `default` configurations are resolved as Draft boosters.

## Product features

### Catalog and watchlists

The catalog separates packs from boxes and supports searching by set name, set code, or product type. Sorting includes release date, EV, the difference between EV and sealed price, skewness, and recent EV movement.

Authenticated users can save products to an account-owned watchlist. Free accounts can save five products, with a replacement flow when full; paid accounts have no application-level watchlist cap.

### Product analysis

The analysis page combines saved product metrics with an optional, on-demand simulation:

- **Value and price:** weighted mean EV, tracked sealed-product price, and the dollar difference between them.
- **Opening outcomes:** median, middle 50% range, a sealed-price comparison when available, and the 99th-percentile value.
- **Simulation inputs:** market, product count, and a card-value cutoff that excludes lower-value cards from the modeled total.
- **Distribution and rare outcomes:** a percentile chart ending at p99, with p99.9 and the highest simulated outcome reported separately.
- **Historical metrics:** EV, sealed price, opening ROI percentage, and skewness for the selected product.

Card-value calculations support TCGPlayer retail prices and Card Kingdom buylist prices. These represent different valuation bases. Tracked sealed prices and comparison charts currently use TCGPlayer data.

The interface also includes representative high-value card imagery and an animated pack-opening reveal. The reveal is a presentation feature; the analytics come from the backend model.

### Historical comparisons

The paid Compare workspace plots two to five packs or boxes using saved daily observations. Users can search for products, filter to saved products, fill the selection from their watchlist, and inspect recorded dates with a pointer, touch, or keyboard.

| Metric | How Compare displays change |
| --- | --- |
| Sealed price | Percentage change from the shared starting date |
| Opening ROI | Percentage-point change in `(card EV − sealed price) / sealed price × 100` |
| Variability | Percentage change in skewness, using the absolute starting value as the denominator |

Comparisons require a shared valid starting date and sufficient recorded observations. Missing prices are not estimated. Lines connect observations across gaps, while date inspection reports recorded values only. Products can have different latest observation dates.

The workspace offers 1W, 1M, 3M, and All ranges, preserves its selection in the URL, and supports up to five account-owned comparison presets. Presets store product selections, the metric, the time range, and an optional label; users can update, rename, replace, or delete them.

### Accounts and interface modes

Auth0 provides sign-in, with server-side token validation and locally stored account entitlements. Longer history, Compare data access, and comparison presets are protected by backend account checks.

The free history endpoint returns the latest seven saved snapshots. Paid users can access all saved history, which supports the analysis page's longer-history controls and the Compare workspace. These are implemented access tiers; this showcase does not demonstrate a billing or checkout system.

The app includes guided onboarding with account-persisted walkthrough status and two interface modes: the standard visual interface and a Terminal-style interface. Mode preference is retained between visits. Compare is available in the standard interface.

## How the analytics work

### Weighted expected value

Each booster configuration specifies weighted variants, slot contents, card pools, and foil treatments. The calculation combines these weights with card prices rather than assuming that every card in a set is equally likely.

EV is the modeled value of tracked cards under the selected market and cutoff. It is calculated separately from the simulated mean.

### Monte Carlo outcomes

The simulation samples configured booster variants and their weighted card pools, maps sampled cards to prices, and aggregates them into pack or box opening totals.

Simulation volume depends on the task:

| Task | Simulated outcomes |
| --- | --- |
| On-demand distribution chart | 10,000–100,000, scaled by the number of packs being modeled |
| Scheduled pack metrics | 1,000,000 per product |
| Scheduled box metrics | 100,000 per product |

NumPy handles sampling and aggregation. On-demand distributions use an input-derived random seed to stabilize results for the same inputs and underlying data.

The application labels its skewness metric **Variability**. It uses the Fisher–Pearson coefficient calculated by SciPy to describe how lopsided the simulated outcomes are. This is not variance or standard deviation, and its rank is not a complete measure of financial risk.

The highest simulated outcome is the largest value observed in that sample, not a theoretical maximum or a guaranteed future result.

### Interpreting ROI

The current interface uses different units in different places:

| Location | Meaning |
| --- | --- |
| Catalog and analysis summary | `EV − sealed price`, in dollars |
| Single-product historical ROI | `(EV − sealed price) / sealed price × 100`, in percent |
| Compare ROI trend | Change in that historical percentage, in percentage points |

For example, a move from −40% opening ROI to −25% appears as **+15 percentage points** in Compare.

These values describe modeled card value before fees and shipping. They do not represent realized sale proceeds.

## Architecture

```text
MTGJSON booster configurations and card prices
                  │
          Scheduled data pipeline
                  │
        Local datasets and SQLite
                  │
       ┌──────────┴───────────┐
       │                      │
Weighted EV engine    Monte Carlo simulation
       │                      │
       └──────────┬───────────┘
                  │
              Flask API ───── Auth0 validation
                  │                   │
                  │          Account-owned SQLite data
                  │          Watchlists · presets · onboarding
                  │
       Browser: HTML / CSS / JavaScript
       Catalog · analysis · history · Compare
```

The browser handles navigation, account state, controls, chart rendering, and historical comparison transformations. Booster modeling, weighted EV, simulation, persistence, and access checks remain backend responsibilities.

| Layer | Technology and responsibility |
| --- | --- |
| API | Python, Flask, JSON responses |
| Analytics | NumPy sampling and aggregation; SciPy skewness |
| Storage | SQLite for EV tables, current metrics, historical snapshots, and account data |
| Interface | HTML, CSS, JavaScript, SVG charts, standard and Terminal modes |
| Identity | Auth0, PyJWT token validation, local account entitlements |
| Data and imagery | MTGJSON booster/card-price data; Scryfall card imagery |
| Runtime | Docker Compose, Gunicorn backend, Nginx frontend |

A GitHub Actions workflow is configured to refresh external datasets and regenerate saved metrics daily. This configuration does not guarantee that every product has a fresh observation on every date.

Analytical databases are mounted read-only into the application container. Account-owned data uses a separate writable persistent volume so watchlists, presets, and onboarding state can survive container replacement.

## Data boundaries

The interface distinguishes current metrics, historical fallback metrics, and unavailable data. It exposes observation dates instead of treating the presence of a database row as proof of freshness.

Sealed prices must be positive to support price-dependent metrics. Missing prices remain unavailable. Card-price lookup can fall back to earlier recorded prices; cards with no usable price contribute zero under the current model.

Custom simulations use the selected inputs, while saved metrics and history reflect the inputs used when those snapshots were generated. Changing a simulation control does not rewrite historical observations.

## Interface screenshots

These screenshots show an earlier interface from September 2026. They document the project's development and do not show the current comparison, watchlist, onboarding, or Terminal workflows.

### Earlier set selection

<img width="1365" height="868" alt="Earlier MTG EV Analyzer set-selection interface, September 2026" src="https://github.com/user-attachments/assets/77415a1d-8ae0-4e96-830b-094649a5f013" />

### Earlier product analysis

<img width="1419" height="793" alt="Earlier MTG EV Analyzer product-analysis interface, September 2026" src="https://github.com/user-attachments/assets/b6583e31-0191-44ee-8e70-e7c3a95f1c58" />

## Why I built it

Sealed-product EV combines structured booster collation, unequal card probabilities, different treatments, and heavily skewed prices. A useful analysis needs to explain both the average and what individual openings can look like.

Pack Oracle brings those questions into one application, combining probability modeling, numerical simulation, a data-refresh pipeline, authenticated APIs, persistent account features, and interactive visualizations.

## Source availability

The full application is maintained in a private repository. This public showcase explains the implemented product and its engineering; it does not include the application source, installation instructions, or a hosted demo.

Magic: The Gathering and related properties are owned by Wizards of the Coast. This project is unofficial and is not affiliated with or endorsed by Wizards of the Coast, MTGJSON, or Scryfall.
