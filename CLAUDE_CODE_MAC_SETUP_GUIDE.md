# Claude Code Agent Runbook: Install and Verify the Acquisition-First Put System on macOS

## Mission

Help the user set up and run the existing `acquisition_first_put_system_v3_mac_ready.zip` project on their Mac, without disabling macOS security protections, leaking API credentials, or submitting trades. The user encountered the macOS warning that Apple could not verify a `.command` launcher. Prefer running inspected shell scripts directly from Terminal over bypassing Gatekeeper.

**Do not rewrite the strategy or claim that demo quotes are live. Do not place orders. Ask the user before installing system-wide software or making material changes.**

## Locate the project

1. Check for an already-extracted directory, normally `~/Downloads/acquisition_first_put_system` (the archive contains that top-level folder). Ask for the path if not found; do not guess or search private locations broadly.
2. If only the ZIP exists, extract it to a user-approved folder, keeping source files intact.
3. Work from the folder containing `pyproject.toml`, `requirements.txt`, `setup_mac.sh`, `put_seller/`, and `config/config.yaml`.
4. Review `README.md`, `MAC_QUICK_START.md`, `setup_mac.sh`, `run_demo_mac.sh`, `run_live_mac.sh`, and `pyproject.toml` before executing scripts. Check any scripts for unexpected destructive commands, untrusted downloads, shell commands requiring administrator privileges, or calls to unknown endpoints. Show the user any concerns rather than executing them.

## macOS security warning

The `.command` files are unsigned/not notarized, which can trigger Gatekeeper. **Do not** disable Gatekeeper (`spctl --master-disable`), globally remove quarantine attributes, run `sudo` to bypass the warning, or promise the archive is malware-free. Instead, inspect project scripts and execute the corresponding `.sh` scripts using Terminal from the project directory. If macOS also blocks a script or installation, stop and explain the precise error; ask the user before any file-specific security override.

## Prerequisites and isolated installation

From the project root, inspect the installed Python version:

```bash
pwd
uname -s
uname -m
python3 --version
```

Python **3.10 or newer** is required. If missing, ask the user before installing Python (for example, using the official python.org installer or Homebrew). No global Python changes are needed when a suitable version is already present.

The project's own setup script is:

```bash
sh ./setup_mac.sh
```

It checks Python, creates a project-local `.venv`, installs dependencies from `requirements.txt`, installs the local project in editable mode, creates `.env` from `.env.example` if needed (preserving existing `.env`), and runs the test suite. `pip` will need Internet access to download packages. If the setup script has any unexpected content, do not run it; explain and use transparent, equivalent commands instead. **Do not use `sudo pip`.**

Verify setup independently:

```bash
.venv/bin/python --version
.venv/bin/python -m pytest -q
```

Expected from the shipped package: **11 tests**. If a test fails, report exact failures, diagnose, and do not say setup completed successfully.

## First demo run (no credentials)

```bash
sh ./run_demo_mac.sh
```

This uses **synthetic** market prices and option chains and a fixed demo as-of date (`2026-10-21`); it does **not** access live market quotes. Verify these files exist:

- `output/latest/decision_report.html`
- `output/latest/trade_shortlist.csv`
- `output/latest/best_by_ticker.csv`
- `output/latest/all_candidates.csv`
- `output/latest/portfolio_allocation.csv`
- `output/latest/run_summary.json`

Use `open output/latest/decision_report.html` if the script did not open it automatically. Inspect the summary and report any exceptions or errors to the user.

Optional position-management demo:

```bash
sh ./run_management_demo_mac.sh
```

Verify `output/management_latest/position_management_report.html` and `output/management_latest/position_actions.csv`. This demo also uses synthetic/example positions.

## Live market data: credentials and watchlist

Do not ask the user to paste API keys into chat or commit keys to Git. Credentials are stored in project-root `.env`:

```dotenv
TRADIER_TOKEN=replace_me
TRADIER_BASE_URL=https://api.tradier.com/v1
ALPHAVANTAGE_API_KEY=replace_me
```

Tell the user where to edit `.env` locally, or open it in their preferred editor with their permission. Never echo token contents, log them, include them in a generated report, or send them to unrelated services. Ensure `.env` is ignored by Git. Tradier supplies stock prices, option chains, Greeks, IV, volume and open interest; Alpha Vantage supplies upcoming earnings dates. Live mode needs valid credentials and sufficient API access; do not assume the sandbox feed is live.

The live watchlist is:

`data/watchlist_live_template.csv`

Explain and help the user populate the existing columns (do not replace headers or invent prices):

- `ownership_approved`: would the user genuinely want to own this company?
- `thesis_intact`: does the investment thesis still hold?
- `target_entry_price`: the **maximum acceptable net assignment basis**.
- `desired_shares`, `max_shares`, `max_contracts`: sizing constraints.
- `manual_news_clear`: manually reviewed current company-specific news.
- `earnings_mode`, `earnings_trade_approved`, `earnings_bear_case_price`: controls for deliberately selling through earnings.
- `min_30d_roi` and any earnings-specific hurdle fields: opportunity thresholds.

Do not automatically set `ownership_approved`, `thesis_intact`, `manual_news_clear`, or `earnings_trade_approved` to true; these are the user's investment judgments. Unknown or unavailable earnings dates should be handled conservatively in accordance with the program's configured gates.

Review `config/config.yaml`, especially:

- `active_profile: acquisition_core` (regular puts, around 30 days to expiration).
- `performance.target_30d_roi: 0.020` (roughly **2% maximum gross 30-day-equivalent premium yield**, not a guaranteed return).
- `performance.include_collateral_interest: false` (user initially chose to exclude money-market interest).
- `capital.total_cash` (currently a demo placeholder; ask for actual intended secured cash before live allocation advice).
- `earnings_acquisition` (a **separate** workflow; requires express approval and a bear-case target price).

The entry-price requirement is `strike - estimated option premium <= target_entry_price`. A high premium must not override this valuation gate. For earnings trades, compare net basis with the user's post-earnings bear-case price. Delta and IV are diagnostics, not substitutes for valuation or approval to own shares.

## Live screening and verification

Only after credentials and the watchlist are in place:

```bash
sh ./run_live_mac.sh
```

Review the exit status, check the timestamp in `output/latest/run_summary.json`, and verify that the dashboard/CSV were generated by the current **live** run, not left over from demo. The live script writes to `output/latest/`, which demo also uses: do not confuse the two. Verify quote timestamps and API data freshness before treating any `SELL` candidate as actionable. If API errors occur, explain the provider, HTTP status/error, and whether the data are delayed, missing, or rate-limited, without exposing credentials.

The key outputs are:

| File | Meaning |
|---|---|
| `output/latest/decision_report.html` | Main visual dashboard, opportunity decisions and rationale |
| `output/latest/trade_shortlist.csv` | One selected put per company |
| `output/latest/best_by_ticker.csv` | Detailed best contract and features for each ticker |
| `output/latest/all_candidates.csv` | Full evaluated option universe |
| `output/latest/underlying_snapshot.csv` | Price, trend, ATR, IV and earnings inputs |
| `output/latest/portfolio_allocation.csv` | Proposed secured-cash allocations |
| `output/latest/run_summary.json` | Run configuration, decision counts and summary |

`SELL` means candidate for **manual review**, never an instruction for automated trade placement. Other decision states can include `WATCH`, `NO TRADE`, `REJECT`, or `ERROR`. Quotes use an estimated fill; recheck the actual broker bid/ask and choose a limit price at execution time. The program does not connect to brokerage accounts to place orders.

## Existing-position management (optional)

If the user already holds short puts or assigned stock, edit `data/open_positions_template.csv` while preserving headers and enter actual position details. Then run:

```bash
sh ./run_management_live_mac.sh
```

Results go to `output/management_latest/position_management_report.html` and `position_actions.csv`. Covered-call recommendations are conditional: a call may limit stock recovery and is not a guarantee of recovering assignment losses. Verify actual fills, basis, news and acceptable exit price manually.

## Suggested operating schedule

- Run a normal live opportunity scan **once each trading day around 10:30–11:30 a.m. Eastern**.
- On a meaningful pullback, optionally rerun around **2:30–3:30 p.m. Eastern** to assess price stabilization and refreshed option quotes.
- Run position management daily when positions are open, and more attentively near expiry or after material news.
- Update intrinsic-value targets and ownership approval after earnings or a fundamental thesis change.
- Do not schedule background tasks or start automatic brokerage execution without explicit user approval.

## Troubleshooting

- **Apple cannot verify `.command`:** use `sh ./setup_mac.sh` and `sh ./run_demo_mac.sh` in Terminal after inspecting source, rather than bypassing security settings.
- **Wrong folder:** `pwd`; ensure `pyproject.toml` and `put_seller/` are present.
- **Python missing/old:** check `python3 --version`; install a supported version with permission.
- **`ModuleNotFoundError`:** use `.venv/bin/python`, not system Python; repeat isolated setup if necessary.
- **Credentials rejected:** confirm `.env` values locally, endpoint permissions, and provider subscription—never display secrets.
- **No `SELL` results:** this may be valid (valuation, earnings, ROI, IV, or news gates); do not loosen user risk constraints automatically.
- **Old/stale dashboard:** verify the newly generated run timestamp and whether the latest run was demo or live.
- **Mac architecture:** the source is Python and should work on Apple Silicon and Intel as long as dependencies install successfully.

## What to report back to the user

Provide a short completion summary with: project path, Python version, test pass/fail count, demo result status, live readiness (`configured`, `needs keys`, or `blocked`), outputs location, and exact follow-up inputs the user must supply. Avoid exposing credentials or implying the system guarantees the 2% monthly objective or 20% after-tax annual return.
