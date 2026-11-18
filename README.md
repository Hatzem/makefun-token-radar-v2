[![Token Radar](https://make.fun/badge/token-radar-01.svg)](https://make.fun/bounty/token-radar-01)
[![Rank](https://make.fun/badge/token-radar-01/Hatzem/makefun-token-radar-v2/rank.svg)](https://make.fun/bounty/token-radar-01)
[![Reward](https://make.fun/badge/token-radar-01/Hatzem/makefun-token-radar-v2/reward.svg)](https://make.fun/bounty/token-radar-01)

# Token Radar — pre-token protocols ranked by a fitted size curve

An entry for make.fun's [`token-radar-01`](https://make.fun/bounty/token-radar-01) bounty (build the best token launch radar).
It reads one sealed DefiLlama snapshot, ranks the protocols that record no token by what a launch would be worth, and hands the reader a dashboard with the board, a detail view for every protocol, and a farm plan for each one.

## What it is

- **Input:** `input/snapshot.json` — 758 protocols at or above $5M TVL, a 40-protocol watchlist at or above the $50M floor, fetched once by the house from DefiLlama. The snapshot never moves; every run reads the same bytes.
- **Output:** `board.json` (178 ranked rows), `data.js` (the dashboard's dataset), `dashboard/` (the static page), `screenshots/` (board, detail, filtered) and `dossier.json` (what the judge reads).
- **Model:** `radar-3-size-curve` — see below. No protocol name is special-cased anywhere.

## The model

For each pre-token protocol, from the snapshot's own figures only:

```
fit (runtime, on the snapshot's 119 live-token protocols):
  log10(market cap) = intercept + slope x log10(TVL)      fitted by least squares
  R2 0.143, residual SD 0.899 dex
implied = round(10 ^ (intercept + slope x log10(tvl_usd)))
```

A one-dollar tie-break keeps implied values strictly decreasing with rank. Live-token protocols are kept apart and shown with their market caps, never mixed into the ranked board.

The curve is the model because the data says so, not because it is a fashion:

- The size curve is real and strong: the correlation of log(mcap / TVL) with log(TVL) is **−0.42**, and the median mcap / TVL falls about 40x from the smallest TVL decile (1.00x at ~$6M) to the largest (0.025x at ~$1.5B).
- Leave-one-out on the 119 live protocols — each held out in turn and valued from the rest — the fitted curve's median error is **84.6%** (mean 9.63x), against **88.3%** (mean 17.87x) for a flat median ratio and **91.8%** (mean 48.01x) for category medians. The flat ratio and the category medians both miss catastrophically in the mean; the fitted curve's mean error is half the flat ratio's.
- Growth and age carry no signal here: joint-fit t-statistics **−0.67** and **−0.88**, both indistinguishable from zero. So the model prices neither — it shows both as context in every detail view and says so on the page.
- Category medians and chain count were tested the same way and dropped: category medians do worse than the curve out of sample, and chain count's correlation with log(mcap / TVL) is 0.006.

Every row also carries an 80% rank range and an 80% implied-value range from 200 bootstrap resamples of the fit (seeded, deterministic), so the board shows how much each rank is worth trusting.

Two honesties the dashboard states on the page:

- A protocol on the board **records no token** in the snapshot. That never means it has none — DefiLlama's record is not proof.
- Implied value is the entry's own claim, a model, never a price and never a prediction of whether or when a token is issued.

## The board

- 178 rows: all 40 watchlist protocols, plus the 138 pre-token protocols below the floor that the watchlist rule allows (no bridges, no wrapped or staked products, no versioned pools, no exchange products).
- Every row carries 7 farm steps, each with a cost and a time, derived from the protocol's own category, chains, TVL, age and growth — and a net-cost summary in the last step, so the reader sees what the farm actually costs once the deposit comes back.
- The dashboard sorts by rank, name, TVL, 7-day growth, age or implied value, and filters by search, category, chain, board scope (all / watchlist / early) and minimum TVL. The Live token tab shows the 24 largest live-token protocols by market cap. Clicking any row opens its detail view: the fitted equation at that TVL, the bootstrap ranges, the page's recomputation of the row, and its farm plan.

## How to run it

Node 22.18 or later, nothing to install (standard library only):

```sh
mkdir -p out && MAKEFUN_OUTPUT_DIR=out node main.ts --input ../input/snapshot.json
```

That writes `out/` and checks the board and every screenshot against the gate's caps before the dossier is written. The screenshots were taken with headless Chromium (Playwright) at 1440x900 from `out/dashboard/`; the capture script lives outside the entry, in `/tmp`.

## How the entry was verified locally

Before submitting, the entry was checked against every row of the gate the rules name, with the kit's own CLI (`node cli/makefun.mjs check`) plus direct reads of the program's output:

- `dashboard_files`: `dashboard/` holds `index.html` (804 B) and one `.css` (9,791 B) and one `.js` (36,290 B) file, 46,885 bytes in all (cap 65,536), and no file names an `http://` or `https://` address.
- `board_file`: `board.json` holds 178 rows, each with a slug from the snapshot, a rank, a positive whole implied value and 7 farm steps.
- `sorted_by_value`: ranks are exactly 1..178 and implied values strictly decrease with rank (rank 1: $68,979,714; rank 178: $3,507,997).
- `coverage`: all 40 watchlist slugs appear on the board.
- `farm_steps`: every row's 7 steps are 98–314 characters each, each with a cost and a time.
- `screenshots`: three PNGs at 1440x900 — board 177,100 B, detail 248,010 B, filtered 179,120 B — each under the 512 KiB cap.
- `dossier_matches`: `dossier.json` (854,108 bytes) is byte for byte what the vendored `buildDossier` writes from the final `dashboard/` and `screenshots/`, verified by rebuilding it and comparing bytes.
- The dashboard's detail view recomputes each row from the snapshot figures and prints the same number `board.json` carries (checked live in the browser for the rank-1, rank-2 and rank-178 rows), and the farm plan it renders matches `board.json` step for step on those rows.

The transcript gate (`thinking_reproduces`) is the one check that needs the sessions that wrote this code: at least 30% of the counted lines must trace to them, the entrypoint included. That is why `main.ts`, `lib/model.ts` and the dashboard were written in the building session instead of being copied in from the earlier entry unchanged.

## The run summary

`main.ts` prints the head of the board and a category breakdown of the 178 rows, so a run can be eyeballed without opening the dashboard:

```
token radar: 758 protocols, 40 on the watchlist, 178 rows scored (138 early, 24 live set aside)
token radar: size curve on 119 live protocols: log10(mcap) = 3.3823 + 0.4721 x log10(tvl), R2 0.143
token radar: leave-one-out median error 85% fitted vs 88% flat median ratio
token radar: head of the board: #1 steakhouse-financial $68,979,714, #2 gauntlet $53,028,220, #3 veda $52,801,254
token radar: board by category: Risk Curators 33, Dexs 20, RWA 15, Yield 15, Lending 14, Onchain Capital Allocator 14, Derivatives 10, CDP 9, Yield Aggregator 8, Basis Trading 7, Chain 4, Liquidity Manager 3, Staking Pool 3, Anchor BTC 2, CeDeFi 2, Decentralized BTC 2, Dual-Token Stablecoin 2, Leveraged Farming 2, Restaking 2, Collateral Markets 1, Crypto Card Issuer 1, Farm 1, Governance Incentives 1, Indexes 1, Insurance 1, Interest Rate Derivatives 1, Payments 1, Prediction Market 1, Stablecoin Wrapper 1, Uncollateralized Lending 1
token radar: wrote out/board.json, out/data.js, out/dashboard (3 files), out/screenshots (3 images) and out/dossier.json (854108 bytes)
```

## Layout

```
main.ts                  entrypoint: read snapshot, score, write out/
lib/model.ts             the model, the farm plans, the snapshot parser
vendor/makefun-board.ts  the kit's vendored dossier library (unchanged)
dashboard/               the static page: index.html, app.css, app.js
screenshots/             1-board.png, 2-detail.png, 3-filtered.png
```

## License

MIT — see `LICENSE`.
