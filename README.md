# DCF Valuation Engine — Forward & Reverse

<!-- ⚠️ BEFORE COMMITTING: replace YOUR_GITHUB_USERNAME and YOUR_REPO_NAME in the
     badge link below with your actual GitHub username and this repo's name,
     or the "Open in Colab" button will 404. -->
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/YOUR_GITHUB_USERNAME/YOUR_REPO_NAME/blob/main/dcf_valuation_engine.ipynb)
![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)

A single discounted-cash-flow engine that values a public company in **two directions from one model**.

**Forward:** project unlevered free cash flow under a growth path that starts at an assumed initial rate and **fades linearly to the terminal rate over ten years**, with EBIT margin converging to its long-run historical level, a CAPM-based WACC built on a **beta estimated by OLS regression** of weekly returns against SPY, and dual terminal-value methods (Gordon growth and exit multiple). Output: intrinsic value per share, scenario ranges, and a WACC × terminal-growth sensitivity grid.

**Reverse:** hold the current market price fixed and solve — with a bracketed root-finder — for the growth path the market is implicitly pricing in. Output: the implied initial growth rate **and** the implied ten-year horizon CAGR, benchmarked against the company's historical revenue CAGR to quantify the *expectations gap*.

The design point: forward and reverse are **not two models**. They are one valuation function solved for different unknowns — forward fixes growth and solves for value; reverse fixes value (the price) and solves for growth. The reverse direction is a Brent-method root-finder wrapped around the identical forward function, which is why the notebook can verify itself with a round-trip test on every run.

## Example finding

<!-- ⚠️ These numbers are from a development run (AAPL, mid-2026 data). Re-run the
     notebook and replace them with your own current output before publishing. -->
Running on **AAPL**: base-case intrinsic value ≈ **$129** vs. a market price of **$282** — but the more interesting output is the reverse read: to justify the market price, the model implies a **~15% compound revenue growth rate sustained for ten years**, against a historical revenue CAGR of **~2%** (4-year log-linear window). The forward number says what the model thinks the stock is worth under conservative assumptions; the reverse number says what the *market* must believe. The ~13-point-per-year gap between implied and historical growth is the debate — and quantifying that debate, rather than issuing a naive over/undervalued verdict, is the purpose of the tool.

## Key features

- **Two-stage fading growth path** — year-1 growth fades linearly to the terminal rate by the final explicit year, so there is no growth cliff at the terminal-value handoff. The 10-year horizon also shifts value out of the terminal period (TV ≈ 50–55% of EV in testing, vs. ~75% for a typical 5-year constant-growth model).
- **Margin convergence** — EBIT margin drifts from the current level to the multi-year historical average rather than assuming today's (possibly peak) margin persists forever.
- **OLS-estimated beta** — regresses 5 years of weekly stock returns on SPY; reports the slope (beta), R², and sample size. Yahoo's beta is kept only as a labeled fallback.
- **Incremental working capital** — the FCF drag is charged as ΔNWC per dollar of revenue *change* (median of yearly ratios from balance-sheet history), not the NWC level — with a labeled fallback when history is too thin to trust.
- **Dual terminal values** — Gordon growth and EV/EBITDA exit multiple, for triangulation.
- **Reverse DCF with honest units** — reports both the implied *initial* growth and the implied *horizon CAGR*; only the latter is compared to historical CAGR, because both are compound multi-year rates. A WACC ±1% table shows how sensitive the implied growth is to the discount-rate assumption.
- **Self-validation on every run** — a built-in cell (1) feeds the forward output back through the reverse solver and confirms it recovers the input growth to within 1e-4, and (2) numerically verifies the value function is strictly monotonic in growth across the solver bracket — the property that makes "the market-implied growth" a unique, well-defined number.
- **Defensive data layer** — alias-tolerant lookups against Yahoo Finance's shifting schema, FRED risk-free rate with two access paths and a documented fallback, a data-quality gate, and a manual-input cell if the automated pull fails.

## Methodology (brief)

Unlevered free cash flow each year: `FCF_t = EBIT_t·(1−τ) + D&A_t − Capex_t − ΔNWC_t`, where revenue growth `g_t` fades linearly from `g_start` to `g_terminal` and margin follows its own linear path to the long-run level. Enterprise value is the sum of discounted explicit-period FCF plus the discounted terminal value (Gordon: `FCF_N·(1+g_T)/(WACC−g_T)`; or exit: `multiple × EBITDA_N`). Equity value = EV − net debt; per-share value divides by shares outstanding. WACC uses CAPM cost of equity (`r_f + β·ERP`) with the OLS beta, after-tax cost of debt inferred from interest expense where sane, and market-value weights.

The reverse direction defines `f(g_start) = value(g_start) − market price` and finds its root with `scipy.optimize.brentq`. Because every year's growth increases with `g_start`, value is monotonic in `g_start`, so the root — the market-implied growth — is unique whenever the price is reachable within the bracket. If the price is *not* reachable, the solver reports that explicitly rather than returning a wrong number: it means growth alone can't reconcile the price under the current margin/WACC assumptions, which is itself information.

## How to run

**Colab (recommended):** click the badge above (after fixing the placeholder), or go to [colab.research.google.com](https://colab.research.google.com) → GitHub tab → paste this repo's URL → open `dcf_valuation_engine.ipynb` → Runtime → Run all. Change `CONFIG["TICKER"]` in the configuration cell to value any company.

**Local:**
```bash
pip install -r requirements.txt
jupyter notebook dcf_valuation_engine.ipynb
```

A FRED API key is optional (`CONFIG["FRED_API_KEY"]`); without one the notebook uses a second no-key access path and then a documented fallback constant. **Never commit a real API key** — keep the committed value as `""`.

## Repository structure

```
.
├── dcf_valuation_engine.ipynb   # the engine (forward + reverse + validation)
├── assets/                      # exported charts for this README
├── requirements.txt
├── CHANGELOG.md                 # v1 → v2: what was wrong and what was fixed
├── LICENSE
└── README.md
```

<!-- ⚠️ After running the notebook, export two charts and drop them in assets/,
     then uncomment these lines:
![Synthesis — your view vs the market's view](assets/synthesis.png)
![Reverse DCF — price vs implied growth](assets/implied_growth_curve.png)
-->

## Limitations (read before quoting results)

Yahoo Finance typically returns only ~4 years of annual statements, so the historical CAGR — even estimated by log-linear regression — is a narrow-window figure; the notebook labels the window on every printout for exactly this reason. The market-implied growth is conditional on the WACC and margin assumptions being the market's too; the WACC-sensitivity table exists to size that dependence rather than hide it. The linear fade is one choice of path shape — the implied *initial* rate would differ under other shapes, which is why the horizon CAGR is the headline number. The incremental-NWC ratio estimated from a handful of yearly observations is noisy and falls back to a labeled level proxy when history is insufficient. And none of this is investment advice: the tool quantifies an expectations debate; it does not settle it.

## Changelog

v2 fixed a set of weaknesses identified in a structured self-review of v1 — including a conceptual error in the working-capital driver and a fragile endpoint-based CAGR. See [CHANGELOG.md](CHANGELOG.md) for the full list.

## Development notes

Developed with AI-assisted coding. The modeling decisions — growth-path structure, driver estimation choices, validation design, and the interpretation framework — are my own, and the methodology above is documented at the level I can defend in conversation.

## Author

**Don Montilla** — B.S. Business Economics, UC San Diego · [LinkedIn](https://www.linkedin.com/in/donalfonso/)

Licensed under the [MIT License](LICENSE).
