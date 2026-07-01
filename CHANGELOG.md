# Changelog

## v2.0 — 2026-07

Fixes and upgrades following a structured self-review of v1:

- **Growth path:** replaced the constant 5-year growth rate (which created an unrealistic cliff at the terminal-value handoff) with a 10-year path that fades linearly from the initial rate to the terminal rate. Terminal value's share of enterprise value drops from ~74% to ~50–55% in testing, shifting weight from the least reliable part of the model to the explicit forecast.
- **Margins:** EBIT margin now converges linearly to the long-run historical average instead of holding the current (possibly peak) margin flat forever.
- **Working capital (conceptual fix):** v1 used the NWC *level* as a percentage of revenue where a *marginal* ratio belongs. v2 charges incremental ΔNWC per dollar of revenue change, estimated as the median of yearly ΔNWC/ΔRevenue from balance-sheet history, clipped for sanity, with a clearly labeled fallback to the level proxy when fewer than three usable yearly observations exist.
- **Beta:** now estimated directly via OLS of 5 years of weekly returns against SPY (slope = beta; R² and sample size reported). Yahoo's opaque single figure is demoted to a labeled fallback.
- **Historical CAGR:** switched from endpoint-to-endpoint (fragile — whipsaws with whichever two years the data source returns) to a log-linear regression of ln(revenue) on time, with the sample window labeled on every printout. Both figures are printed so the method's impact is visible.
- **Reverse DCF units:** now reports both the implied *initial* growth and the implied *horizon CAGR*. Only the horizon CAGR is compared against historical CAGR — both are compound multi-year rates, so the expectations gap is finally an apples-to-apples number. Added a WACC ±1% sensitivity table for the implied growth.
- **Self-validation:** new runtime cell verifies (1) a forward→reverse round-trip recovers the input growth to within 1e-4 and (2) the value function is strictly monotonic in growth across the solver bracket — the uniqueness guarantee for the implied growth. Failures print loudly before any downstream results.

## v1.0

Initial release: 5-year constant-growth forward DCF with CAPM WACC, dual terminal values (Gordon growth and exit multiple), WACC × terminal-growth sensitivity grid, Brent-method reverse solve for market-implied growth, and a synthesis dashboard (football field + implied-vs-historical growth).
