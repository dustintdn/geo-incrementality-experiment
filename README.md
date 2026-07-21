# Geo Incrementality Experiment: Did the Campaign Actually Work?

## Business Question

A marketing campaign ran in a set of treatment markets while comparable control markets received no spend. Did the campaign drive incremental sales, or would those sales have happened anyway?

## Data

The analysis uses Google's public [matched_markets](https://github.com/google/matched_markets) geo experiment dataset: daily sales for **100 geos** (Jan 5 – Apr 7, 2015) from a designed geo experiment in which **50 treatment geos received $50,000 of ad spend between Feb 16 and Mar 15, 2015**, and 50 control geos received none. Because the spend data is real, the ROAS estimate is grounded in actual campaign cost rather than an assumption. Both CSVs are checked into `data/`, so no download is needed.

## Approach

A **matched-market test** analyzed with **CausalImpact** (Google's Bayesian structural time-series library). The method works in two steps: first, identify control geos whose pre-campaign sales moved in lockstep with the treatment geos; second, use that relationship to forecast what treatment-geo sales *would have been* without the campaign. The gap between observed sales and that counterfactual is the causal estimate of lift — something a simple before/after comparison cannot provide. (In this dataset, the naive before/after comparison suggests a 50.7% "lift"; the causal estimate is less than half that, with the rest explained by trend and seasonality shared with the control geos.)

## Key Findings

- **+22.2% incremental sales lift** in treatment geos during the 4-week campaign window (95% credible interval: **+13.3% to +31.2%** — the interval excludes zero)
- **~$181K cumulative incremental sales** against **$50,000 of actual campaign spend**
- **Incremental ROAS: 3.6×** (95% credible interval: 2.2× – 5.1×)

![Observed vs. Counterfactual Sales](data/fig3_obs_vs_counter.png)

*The blue line shows actual treatment-geo sales; the green dashed line is the model's counterfactual (what sales would have been without the campaign). The shaded blue region is the estimated incremental lift.*

## Method Validation

Two checks, covering both failure modes:

- **Recovery test (simulation):** the pipeline was run on simulated data with known ground-truth lifts of 5%–30% — the only setting where the true answer is knowable. Estimates tracked the truth and the true lift fell inside the 95% credible interval in all five runs.
- **Placebo (A/A) test on real data:** the pipeline was applied to a fake "treatment" group built from real control geos that received zero spend. It correctly found no effect (95% CI: −4.3% to +14.4%, straddling zero). An earlier placebo design with mismatched group composition produced a spurious positive — kept in the notebook as a documented failure mode, since a placebo test is only as valid as the comparability of its two sides.

![Method Validation: Recovery Test](data/fig4_recovery_test.png)

![Cumulative Effect: Real vs. Placebo Test](data/fig5_placebo_test.png)

*Left: cumulative incremental sales climb steadily for the real campaign, with the credible band above zero. Right: the placebo run on untreated geos hovers around zero.*

## Recommendation

The campaign produced a statistically credible lift, and at 3.6× incremental ROAS it clears profitability if incremental gross margin exceeds ~28% of revenue (1/ROAS). The natural next step is a second wave in additional geos with a pre-registered analysis plan — pre/post periods, control-selection rule, and decision threshold fixed before the data comes in.

## How to Run

```bash
git clone https://github.com/dustintdn/geo-incrementality-experiment.git
cd geo-incrementality-experiment

python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt   # installs TensorFlow — expect a few minutes

jupyter notebook notebooks/geo_experiment_analysis.ipynb
```

Run all cells top-to-bottom (`Kernel → Restart & Run All`). Random seeds are fixed, so results reproduce exactly.

## Caveats

- **Spillover risk:** if treatment and control geos are geographically adjacent, campaign exposure may leak into control geos and compress the measured lift. The public dataset anonymizes geo identity, so adjacency can't be verified here; a live test should enforce geographic separation.
- **Match quality:** the counterfactual is only as good as the pre-period relationship between treatment and controls. The notebook reports pre-period correlations and plots the pre-period fit — verify the match before trusting the post-period estimate.
- **Pre-period length:** six weeks captures weekly cycles but not annual seasonality; seasonal businesses need a pre-period spanning a full cycle.
- **External shocks:** any event affecting only treatment geos during the campaign window (local promotions, weather) is indistinguishable from campaign effect. Randomized geo assignment makes this unlikely but not impossible.
