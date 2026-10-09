# Carvana Marketing Analytics: First-Touch Channel Attribution

An analysis of paid marketing channel performance for 2022, built for a Carvana analytics technical assessment. Using **first-touch attribution**, the project ties every sale back to the first paid click a customer made, then measures each channel's economics (CPC, CAC, ROI) and recommends how to allocate a 20% budget increase.

**Tools:** Python (pandas) · SQL (SQLite via jupysql) · Jupyter

---

## Table of Contents

- [Business Problem](#business-problem)
- [Data](#data)
- [Methodology](#methodology)
- [Key Findings](#key-findings)
- [Recommendation](#recommendation)
- [Limitations](#limitations)
- [Repository Structure](#repository-structure)
- [How to Run](#how-to-run)

---

## Business Problem

Carvana buys traffic from multiple paid channels. To plan next year's budget, the marketing team needs to know:

1. Which channels, partners, and campaigns are over- or under-performing?
2. What does each channel cost per click, per sale, and per dollar of profit returned?
3. How should an extra 20% of budget be spent?

## Data

Six CSV files describing the funnel from ad click to sale:

| File | Description | Key columns |
|---|---|---|
| `clicks.csv` | Every paid click that brought a user to the site (~169K rows) | `click_datetime`, `channel_id`, `user_id` |
| `locks.csv` | Vehicle "locks" (purchase initiated, ~14K rows) | `lock_id`, `user_id`, `lock_datetime` |
| `sales.csv` | Completed sales (1,348 rows) | `user_id`, `lock_id`, `sale_id`, `make`, `model`, `has_trade_in`, `is_financed`, `apr`, `delivery_distance` |
| `spend.csv` | Daily spend per channel (Jun to Dec 2022) | `date`, `channel_id`, `spend` |
| `ad_channels.csv` | Channel lookup, named `category-partner_campaign` | `channel_id`, `channel` |
| `vehicles.csv` | Average margin by make/model | `make`, `model`, `bodystyle`, `avg_margin` |

**Channel hierarchy.** `channel_name` is parsed into three levels. If a channel has no campaign suffix, the partner is treated as its own campaign:

```
Search Engine-Hooli_Sedan   ->  category: Search Engine | partner: Hooli | campaign: Sedan
Social Media-KnickKnack     ->  category: Social Media   | partner: KnickKnack | campaign: KnickKnack
```

The data is dummy data provided for the assessment, but the categories, partners, and campaigns mimic real-world structures.

## Methodology

### First-touch attribution
Each user's earliest click determines the channel credited with all of their later locks and sales. A customer who clicked KnickKnack and later DatAnkle gives 100% credit to KnickKnack.

### Profit per sale
Profit is built from the vehicle's average margin, adjusted per sale:

1. **Starting value:** `avg_margin` for the make/model.
2. **APR modifier:** For financed sales, `(sale APR - monthly avg APR) / monthly avg APR`, where the monthly average uses financed sales only. Non-financed sales use `-0.1`.
3. **Modified value:** `starting value x (1 + APR modifier)`.
4. **Bodystyle adjustments:**

| Bodystyle | Adjustment | Trade-in |
|---|---|---|
| Sedan / Hatchback | +200, minus 0.5 x delivery distance | +400 |
| Coupe / SUV | minus 0.8 x delivery distance | +300 |
| Truck | -200, minus delivery distance | +200 |

### Metrics

```
CPC = Spend / Clicks
CAC = Spend / Sales (first-touch)
ROI = Profit / Spend
```

### Data handling notes
- `sales.csv` contains a misspelled make (`NIssan`); make names are normalized before joining to `vehicles.csv`.
- 13 sales have no `user_id`, and 27 more belong to users with no recorded click. These 40 sales cannot be first-touch attributed and are excluded from channel-level figures.
- `sale_datetime` uses `M/D/YYYY H:MM` format and must be parsed explicitly. SQLite's `STRFTIME` returns `NULL` on this format, so profit and APR calculations that need the sale month were done in pandas.

## Key Findings

All figures use first-touch attribution unless noted.

### Performance by category

| Category | Spend | Clicks | Sales | CPC | CAC | Profit | ROI |
|---|---:|---:|---:|---:|---:|---:|---:|
| Finance Partnership | $38.5K | 12,004 | 245 | $3.21 | $157 | $88.8K | **2.31** |
| Search Engine | $119.0K | 94,591 | 604 | **$1.26** | $197 | $200.1K | 1.68 |
| Social Media | $30.0K | 22,797 | 144 | $1.32 | $208 | $45.0K | 1.50 |
| Online Video | $39.5K | 18,483 | 189 | $2.14 | $209 | $50.5K | 1.28 |
| Third Party Listing | $62.5K | 21,032 | 126 | $2.97 | $496 | $17.3K | **0.28** |

### Question-by-question answers

| # | Question | Answer |
|---|---|---|
| 1 | Lowest-CPC category | **Search Engine** at $1.26. Auction-based, high-intent inventory at massive scale, and it also delivers the most clicks by far. |
| 2 | Top 3 campaigns by sales | **Hooli Sedan** (174), **Debit Dharma** (137), **Hooli Convertible** (135). |
| 3 | Lowest-CAC partner | **Debit Dharma** at $146, ahead of Giving Vine ($171) and Hooli ($189). |
| 4 | Best / worst ROI category | **Finance Partnership (2.31)** best, **Third Party Listing (0.28)** worst. |
| 5 | Avg. distinct channels before a lock | About **4.2** channels per user. |
| 6 | Sales with no click in prior 90 days | **51 of 1,348 (3.8%)**. Likely causes: cross-device or cookie tracking gaps, ad blockers, and word-of-mouth or direct visits with no paid click. |
| 7 | Reallocate a 20% budget increase | See [Recommendation](#recommendation). |

### Partner and campaign highlights

- **Finance Partnership is the efficiency leader.** Debit Dharma (ROI 2.43, CAC $146) and Giving Vine (ROI 2.17, CAC $171) are the two best partners on cost per sale, despite high CPCs. Their clicks convert well.
- **A low CPC does not mean a low CAC.** Finance has the highest CPC ($3.21) but the lowest CAC, because its traffic converts at a much higher rate.
- **Bam Sedan is a standout.** At ROI 2.39 and CAC $165 on only $15K of spend, it is the best-performing search campaign. Bam Truck (ROI 1.07, CAC $316) is the weakest.
- **"Teen" video campaigns underperform.** WeTube Teen (ROI 0.23, CAC $833) and DatAnkle Teen (ROI 0.75, CAC $625) are far below their 25+ counterparts (ROI 1.72 and 1.37).
- **Third Party Listing loses money on both partners.** RealTruck (ROI 0.29) and ManualBarter (ROI 0.26) each have CAC above $470, more than 2.4x the next-worst category.

## Recommendation

**Shift budget toward Finance Partnership and Search, and cut Third Party Listing.**

With a 20% increase on $289.5K of 2022 spend (about $57.9K), plus the roughly $62.5K freed from Third Party Listing, there is about $120K to deploy. An illustrative split:

| Category | Change | Rationale |
|---|---:|---|
| Finance Partnership | **+$40K** | Best ROI (2.31) and lowest CAC. It has the smallest base budget, so it has the most headroom before returns diminish. |
| Search Engine | **+$50K** | The volume engine, with the lowest CPC and the largest absolute profit ($200K). Favor Bam Sedan and Hooli Sedan. |
| Social Media | +$15K | Solid, steady ROI (1.50). |
| Online Video | +$15K | Mid-range ROI (1.28). Skew toward 25+ campaigns over Teen. |
| Third Party Listing | **Cut to $0** | ROI 0.28 and CAC above $490. It is the one channel worth exiting, not just deprioritizing. |

Also worth testing: pause the Teen video campaigns and Bam Truck, and re-allocate to their stronger siblings.

## Limitations

- **First-touch bias.** First-touch rewards channels that *start* journeys and gives zero credit to channels that *close* them. Customers touch about 4 channels before locking, so first-touch likely undervalues closing channels. A multi-touch or position-based model would be a useful cross-check.
- **Diminishing returns are not modeled.** The allocation assumes returns stay near current averages. Doubling Finance Partnership spend may not hold a 2.31 ROI, so scale in stages and monitor marginal CAC.
- **Unattributed sales.** 40 sales (about 3%) have no first-touch click and are excluded from channel totals.
- **Profit is a modeled estimate** built from average margins and the rules above, not actual transaction-level profit.
- **Window mismatch.** Spend covers Jun to Dec 2022, while some sales and clicks extend into early 2023.

## Repository Structure

```
.
├── README.md
├── data/
│   ├── ad_channels.csv
│   ├── clicks.csv
│   ├── locks.csv
│   ├── sales.csv
│   ├── spend.csv
│   └── vehicles.csv
└── notebooks/
    ├── 1_instructions.ipynb   # Assessment prompt, profit rules, questions
    ├── 2_workspace.ipynb      # SQL + pandas working code
    └── 3_answers.ipynb        # Final answers and written rationale
```

## Author
**Aditi Somani**
