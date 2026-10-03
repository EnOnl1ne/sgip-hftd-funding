# Plan: Does HFTD Eligibility Predict SGIP Battery Funding?

ECON 630 problem set: "Test an Economic Question with Public Data." It also serves as the
prima facie result for my ECON 676 proposal, and is meant to be useful to Ava Community Energy's
VPP / battery storage outreach.

## 1. Question and expectation

**Question:** Among residential battery storage applicants to California's Self-Generation
Incentive Program (SGIP) in PG&E territory, are applicants located in a High Fire-Threat
District (HFTD Tier 2/3) — which qualifies them for enhanced "Equity Resiliency" incentives —
more likely to reach funded (`Paid`) status than comparable applicants in the same zip code
who are not in an HFTD?

**Expectation:** Yes. HFTD applicants should be paid more often, because the enhanced
incentive covers most or all of the battery cost and draws on a separate budget.
I expect the gap to shrink once application timing and zip code are held fixed, since
pre-coding checks (Section 6) show HFTD applications were submitted earlier on average.

**Secondary (descriptive) question:** Do zip codes with a larger share of HFTD applicants
have more SGIP storage applications per 1,000 housing units?

## 2. Data sources

All data is pulled inside the notebook. No manually downloaded files are used.

| Source | Access | What it provides |
|---|---|---|
| SGIP Weekly Statewide Report | `requests.get("https://www.selfgenca.com/documents/reports/statewide_projects")`. This URL returns the latest `.xlsx` directly. SGIP has no formal API. | Application-level data: administrator, sector, storage kWh, HFTD tier, PSPS flag, zip, `Date Received`, `Budget Classification` (outcome) |
| U.S. Census Bureau ACS 5-year, 2024 vintage (2020–2024) | Census Data API, `https://api.census.gov/data/2024/acs/acs5`, via `requests`, with `for=zip code tabulation area:*` | `B25001_001E` total housing units, `B19013_001E` median household income, by ZCTA |

- **API key:** the Census API requires a key. It is stored as `CENSUS_API_KEY` in a `.env` file,
  loaded with `python-dotenv` (`load_dotenv()` + `os.getenv`). It is never written in the notebook.
- **Reproducibility caveat:** the SGIP URL serves a new snapshot every week, and application
  statuses keep changing. The notebook saves the downloaded file to `data/` (gitignored) under its
  original dated filename (taken from the `Content-Disposition` header) and prints the snapshot
  date, so results can be tied to a specific report. A rerun on a later week will reproduce the
  method, but the numbers may differ slightly.
- **Python packages:** `requests`, `python-dotenv`, `pandas`, `openpyxl`, `matplotlib`, `pyfixest`
  (fixed-effects regression with clustered SEs), `scipy` (two-proportion test).

## 3. Cleaning steps

On the SGIP data (header is on Excel row 4, so `pd.read_excel(..., header=3)`). The notebook
logs the row count after each step:

1. `SGIP Administrator == "Pacific Gas and Electric"`
2. `Host Customer Sector` in `{Residential, Single Family, Multifamily}`
3. `Energy Storage Capacity (kWh)` not null (storage projects only)
4. Drop `Fully Qualified State == "RRF Rejected"` (never reviewed, so no budget assigned)
5. `Program Year >= 2024`: after CPUC D.24-03-071 / AB 209, avoids mixing two eligibility regimes
6. Drop rows with a missing `Located in HFTD` value (~527 rows)
7. Parse `Date Received`. The format is `01/02/24 10:59:54.708805 PST`, so strip the timezone
   label and parse with an explicit format. Keep applications received **before 2026-01-01**,
   because 2026 applications are almost all still unresolved (Paid rate ≈ 0%) and would
   mechanically depress whichever group applied later.
8. Build variables:
   - `paid` = 1 if `Budget Classification == "Paid"`, else 0
   - `hftd` = 1 if Tier 2 or Tier 3, 0 if Not Applicable; plus separate `tier2` / `tier3` indicators
   - `psps` = 1 if `Experienced Two PSPS Events == "Yes"`, else 0
   - `received_quarter` = calendar quarter of `Date Received` (timing control)
   - `zip5` = 5-character string zip

On the Census data:

9. Pull all ZCTAs in one request, keep California ZCTAs (`90000`–`96199`), convert estimates to
   numeric, and set Census missing-value sentinels (e.g. `-666666666`) to `NaN`.
10. Left-merge onto SGIP by `zip5` = ZCTA. Report how many SGIP zips fail to match, since zip
    codes and ZCTAs are not identical geographies (PO-box-only zips have no ZCTA).
11. For the secondary question, collapse to zip level: number of applications, share HFTD,
    applications per 1,000 housing units.

## 4. Charts and tests

**Charts.** Every chart gets a title, axis labels with units, and a source note
("Source: SGIP Weekly Statewide Report, <snapshot date>; U.S. Census ACS 2020–2024").

1. Paid rate (%) by HFTD status (Not Applicable / Tier 2 / Tier 3), bar chart with 95% CIs
2. Paid rate (%) by application quarter, one line per HFTD status: shows the timing confound
3. Paid rate (%) by HFTD status × PSPS status, grouped bars: shows the second eligibility route
4. Coefficient plot: the HFTD estimate across the regression specifications below
5. Zip-level scatter: SGIP storage applications per 1,000 housing units vs. share of the zip's
   applicants in HFTD (secondary question)

**Tests.**

- **Raw gap:** two-proportion z-test of the Paid rate, HFTD vs. Not Applicable.
- **Main specification:** linear probability model,
  `paid ~ hftd + psps | zip5 + received_quarter`, standard errors clustered by zip
  (`pyfixest.feols`). The coefficient on `hftd` is the within-zip, same-quarter difference in the
  probability of being paid, in percentage points. Controlling for `psps` matters because
  non-HFTD households with two PSPS events can also qualify for enhanced incentives. Without it,
  part of the comparison group is effectively treated.
- **Robustness checks:**
  - (a) No zip fixed effects, with zip median household income (ACS) as a control instead,
    to show how much the zip fixed effects matter
  - (b) Tier 2 and Tier 3 entered separately
- Report N, the number of zip clusters, and the number of zips with both HFTD and
  non-HFTD applicants (the identifying variation for the zip fixed effects).

## 5. What would support or contradict the expectation

- **Supports:** the `hftd` coefficient is positive and statistically significant (p < 0.05) in
  the main specification. That means HFTD applicants are more likely to be funded even when
  compared with neighbors in the same zip who applied in the same quarter.
- **Contradicts:** the coefficient is near zero or not significant once zip and quarter
  are controlled for. That would mean the raw gap reflects *when* and *where* HFTD households
  applied, not eligibility itself.
- **Partial support:** a positive `psps` coefficient of similar size would suggest that
  enhanced-incentive eligibility through *any* route, not HFTD specifically, drives funding.
- **For Ava:** if eligibility predicts funding, target VPP/battery outreach by HFTD/PSPS
  geography. If it doesn't, the binding constraint is more likely awareness or application
  complexity than incentive generosity.

## 6. Pre-coding checks (run on the 09/20/2026 snapshot, for planning only)

- Final 2024+ sample before steps 6–7: 9,910 rows (1,771 Tier 2 / 780 Tier 3 / 6,832 Not
  Applicable / 527 missing). Raw Paid rate: 50.1% / 44.4% / 33.9%.
- **Timing confound:** HFTD applications are concentrated in early 2024, when Paid rates were
  highest for everyone. Within half-year cohorts the gap is much smaller
  (e.g. 2025H2: 25% Not Applicable vs. 31% Tier 2).
- **PSPS overlap:** 1,176 of the 6,832 Not Applicable applicants report two PSPS events, and their
  Paid rate (53%) is about the same as Tier 2's.
- 238 zips contain both HFTD and Not Applicable applicants.

## 7. Limitations to address in the conclusion

- Not a causal regression discontinuity. Zip is the finest geography available, so this is
  a within-zip comparison. HFTD households may differ from their neighbors in unobserved ways
  (e.g. more rural, larger lots).
- Part of the HFTD effect may be mechanical: HFTD applicants draw on a separate budget
  (Equity Resiliency) with its own funding availability.
- `Cancelled` is counted as not paid, but it may reflect the household's own choice.
- The SGIP data is a moving snapshot (see Section 2), and zip → ZCTA matching is imperfect.

## 8. Repo and workflow

- New public GitHub repo for this project only, containing `plan.md`, `sgip_hftd.ipynb`,
  `requirements.txt`, `README.md`, and `.gitignore` (which excludes `.env` and `data/`).
- Commit at each stage: plan → data pull → cleaning → charts → regressions → conclusion.
- Submit: repo link, `plan.md` and the notebook exported as HTML, and a short note on how
  Claude Code was used.

## Out of scope (future work for ECON 676)

Logit / conditional-logit estimates; full post-2017 sample with year fixed effects; GIS-based
HFTD intensity weighting using CPUC shapefiles; the Stockton/Lathrop NEM 3.0
difference-in-differences idea.
