# MTG EV Analyzer

A full-stack analytics platform for estimating the expected value and risk profile of Magic: The Gathering sealed products.

**Python · Flask · NumPy · SQLite · JavaScript**

**All 33 years of MTG sets analyzed · 100,000–1,000,000 simulated outcomes per analysis · 5500 lines of source code**

---

## Overview

MTG EV Analyzer models the financial distribution of opening Magic: The Gathering booster packs and boxes.

Rather than reporting only the average card value of a product, the analyzer models the underlying booster composition and card-price distributions to answer questions such as:

- What is the expected value of a pack or box?
- How often does an opening return substantially less than its purchase price?
- What do typical, high-roll, and low-roll openings look like?
- How much of a product's expected value is concentrated in rare outcomes?
- How does the distribution change across different booster types and sets?

The application combines booster configuration and pricing data with exact weighted calculations and Monte Carlo simulation, exposing the results through a Flask API and interactive browser frontend.

## Screenshots

### Set Selection

<img width="1365" height="868" alt="Screenshot 2026-09-03 at 4 17 45 PM" src="https://github.com/user-attachments/assets/77415a1d-8ae0-4e96-830b-094649a5f013" />

The main interface provides searchable set selection and routes the selected product into the analysis workflow.

### Product Analysis

ADD IMAGE

The analysis interface exposes product metrics, booster configuration, representative card hits, and simulation controls.

## Core Features

### Expected Value Analysis

The backend computes weighted expected value from the possible cards and outcomes within each booster slot rather than treating every card as equally likely.

This allows the model to account for differences in:

- rarity
- booster slot composition
- card inclusion probability
- foil and non-foil treatments
- alternate product configurations
- individual card prices

### Monte Carlo Simulation

For metrics that depend on the complete distribution rather than its mean, the application runs configurable **100,000–1,000,000-outcome Monte Carlo simulations** using NumPy.

Simulated pack and box openings are aggregated into value distributions used to estimate:

- median outcome
- value percentiles
- variability - modeled by calculating the fisher-pearson coefficient on the samples
- probability of exceeding selected value thresholds
- upper-tail / jackpot outcomes
- distribution shape beyond simple expected value

This distinction is important because two products with similar expected values can have substantially different risk profiles.

### Set and Booster Coverage

The analyzer supports **150+ Magic: The Gathering sets** and multiple booster configurations.

Booster and pricing information is sourced from **MTGJSON**, while card imagery is retrieved separately for frontend presentation.

### Historical Metrics

Calculated metrics are persisted in **SQLite**, allowing the application to retain both current and historical analytical results instead of recomputing every value from scratch.

## Architecture

```text
                    ┌─────────────────────┐
                    │    Browser Client   │
                    │ HTML / CSS / JS UI  │
                    └──────────┬──────────┘
                               │
                          HTTP / JSON
                               │
                    ┌──────────▼──────────┐
                    │      Flask API      │
                    │ request validation  │
                    │ + orchestration     │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
     ┌────────────────┐ ┌──────────────┐ ┌──────────────┐
     │ EV Calculation │ │ Monte Carlo  │ │    SQLite    │
     │     Engine     │ │  Simulation  │ │ Persistence  │
     └───────┬────────┘ └──────┬───────┘ └──────────────┘
             │                 │
             └────────┬────────┘
                      ▼
             ┌─────────────────┐
             │ MTGJSON booster │
             │ + pricing data  │
             └─────────────────┘
```

The frontend is intentionally separated from the analytics layer. The browser requests data through the Flask API, while booster modeling, weighted calculations, simulations, and persistence remain backend responsibilities.

## Technology

| Layer                   | Technology            |
| ----------------------- | --------------------- |
| Backend                 | Python, Flask         |
| Statistical computation | NumPy                 |
| Database                | SQLite                |
| Frontend                | JavaScript, HTML, CSS |
| Booster / pricing data  | MTGJSON               |
| Card imagery            | Scryfall              |
| API format              | REST / JSON           |

## Implementation

The complete production source for MTG EV Analyzer is maintained in a **private repository**.

This repository serves as a public technical overview of the project rather than a distributable copy of the application. The full implementation includes the booster-generation logic, probability modeling, simulation engine, API layer, persistence system, data-processing pipeline, and frontend application.

Keeping the core implementation private allows me to demonstrate the architecture, engineering decisions, and behavior of the system without publishing the complete application for unrestricted redistribution.

Representative portions of the implementation may be included here where they help demonstrate specific technical decisions without exposing the full system.

## Selected Implementation Details

### Weighted Booster Modeling

A booster is modeled as a collection of slots, with each slot having its own possible card pools and probability distribution.

The analytics layer resolves these slot-level probabilities before computing expected value or generating simulated packs. This avoids approximating a booster as a uniform random selection from an entire set.

### Simulation Pipeline

At a high level, a simulation follows this flow:

```text
Select set and booster type
        ↓
Load booster configuration
        ↓
Resolve card pools and probabilities
        ↓
Sample each booster slot
        ↓
Map sampled cards to market values
        ↓
Aggregate pack / box value
        ↓
Repeat 100,000–1,000,000 times
        ↓
Calculate distribution statistics
```

NumPy is used for the high-volume numerical work so that hundreds of thousands of simulated openings can be evaluated in 1-10 seconds depending on size.

### API / Frontend Boundary

The frontend does not reproduce the analytics logic.

Instead, it handles set and booster selection, asynchronous requests, loading states, card presentation, and visualization while the Flask backend remains responsible for calculation and data access.

This separation keeps the statistical implementation centralized and allows the analytical layer to evolve independently of the interface.

### Persistence

SQLite stores calculated and historical metrics locally. Separating persisted analytical results from the raw external datasets reduces unnecessary recomputation and provides a foundation for comparing changes in sealed-product value over time.

## Project Motivation

I built MTG EV Analyzer because sealed-product expected value is more complicated than averaging the prices of cards in a set.

Booster products have structured collation, unequal card probabilities, multiple slots, different treatments, and heavily skewed price distributions. A product can have a reasonable average value while most individual openings fall substantially below that average.

The project became an opportunity to combine an area I already understood well, Magic: The Gathering, with probability, simulation, backend development, API design, data persistence, and frontend engineering.

## Source Availability

The full application is not publicly distributed. This repository contains documentation, screenshots, architecture information, and selected implementation examples intended to demonstrate the engineering behind the project.

Magic: The Gathering and related properties are owned by Wizards of the Coast. This project is unofficial and is not affiliated with or endorsed by Wizards of the Coast, MTGJSON, or Scryfall.
