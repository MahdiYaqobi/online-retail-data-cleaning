# Online Retail: From Messy Transactions to Business Insights

I took a year of real transactions from a UK online gift shop, cleaned them, checked whether the cleaned data could actually be trusted, and then used it to answer seven business questions. I also tried to be honest about what the data can and can't tell us.

## The short version

**The data:** 541,909 transaction lines from a UK-based online retailer that sells all-occasion gifts, from 1 December 2010 to 9 December 2011 ([UCI Online Retail dataset](https://archive.ics.uci.edu/dataset/352/online+retail)).

**After cleaning and checking:** about **£9.96 million in product sales**, across **19,770 invoices** and **4,333 identified customers**.

### What I found

1. **Sales take off in autumn, and it is about more orders, not bigger ones.** November revenue (about £1.45M) was roughly double August's. The number of orders doubled (1,339 to 2,751) while the average order stayed around £530-565.
2. **Most of that growth came from customers who had already bought before.** About 89% of the rise in revenue from identified customers between August and November came from returning customers.
3. **A small group of customers carries the business.** The top 1% of identified customers (43 people) bring in about 30% of identified revenue, and the top 10% bring in about 60%.
4. **Revenue is spread across many products.** It takes about 22% of products to reach 80% of revenue, and even the ten best sellers add up to only 8%. No single product dominates.
5. **The UK is the business, and the export numbers are fragile.** The UK brings in about 85% of revenue. The Netherlands, Ireland and Australia look big among export markets, but each rests on only 3 to 9 customers (a single Dutch customer accounts for 98% of the Netherlands' revenue). Germany and France, with about 90 customers each, look like the steadier markets.
6. **Orders arrive in the middle of the day.** About 78% of orders are placed between 10:00 and 15:59. Thursday is the busiest day and Sunday the quietest, with smaller orders too. No transactions are recorded on Saturdays.
7. **Cancellations are small.** They amount to roughly 2% of product revenue.

### What this suggests

- Plan staffing, stock and deliveries around the autumn peak and the daytime hours.
- Look after the biggest customers. Losing even a few dozen of them would hurt. The good news: eight of the ten largest had ordered within the last nine days of the data.
- Treat the biggest export "markets" as a handful of accounts rather than real markets until we know more about who they are.

### What I'd like you to take from this project

- **I check the data before I trust a number.** While auditing the cleaned data I found three huge orders (worth £284,623, or 2.8% of revenue) that were cancelled within minutes. Left in, they would have inflated revenue and made the cancellation rate look twice as high (4.6% instead of 2.3%).
- **I write down my assumptions and limits** instead of hiding them. There is a full [Limitations and assumptions](#limitations-and-assumptions) section below.
- **I keep the counts honest.** Every cleaning step reconciles back to the 541,909 raw rows, and the numbers quoted above are recomputed in an appendix at the end of the analysis notebook.

---

*Everything below is for readers who want the details: how the data was cleaned, how the analysis was done, and how to run it yourself.*

## Contents

- [About the data](#about-the-data)
- [How the project is organized](#how-the-project-is-organized)
- [Stage 1: Cleaning the data](#stage-1-cleaning-the-data)
- [Stage 2: Analysis](#stage-2-analysis)
- [Limitations and assumptions](#limitations-and-assumptions)
- [Run it yourself](#run-it-yourself)
- [Project structure](#project-structure)
- [What's next](#whats-next)
- [Credits and license](#credits-and-license)

## About the data

The [Online Retail dataset](https://archive.ics.uci.edu/dataset/352/online+retail) comes from the UCI Machine Learning Repository. It contains every transaction made between **1 December 2010 and 9 December 2011** by a UK-based and registered online store that sells all-occasion gifts. According to the dataset's documentation, many of its customers are wholesalers, which explains some of the very large order quantities you'll see.

It has **541,909 rows** and 8 columns:

| Column | What it is |
|---|---|
| `InvoiceNo` | Invoice number. A leading `C` means the order was cancelled. |
| `StockCode` | Product code |
| `Description` | Product name |
| `Quantity` | Units per transaction |
| `InvoiceDate` | Date and time of the transaction |
| `UnitPrice` | Price per unit (in pounds sterling) |
| `CustomerID` | Customer identifier |
| `Country` | Customer's country |

One thing to keep in mind from the start: each row is a **product line**, not an order. One invoice (an order) can have many lines, and one customer can have many invoices. That's why the analysis counts distinct invoices when it talks about orders.

## How the project is organized

The work has two stages, and each stage has its own notebook:

| Notebook | What it does |
|---|---|
| `notebooks/data_cleaning.ipynb` | Stage 1. Turns the raw Excel file into two clean CSVs: sales and returns. |
| `notebooks/eda_notebook.ipynb` | Stage 2. The main notebook. Audits the cleaned data, then answers the business questions. Starts with a one-page executive summary. |
| `notebooks/returns_notebook.ipynb` | An early look at the returns file. I kept it for reference, but the final returns analysis lives in section 4.3 of the EDA notebook. |

## Stage 1: Cleaning the data

### What was wrong with it

Before changing anything, I profiled the data. Here is what stood out:

- **Missing `CustomerID` on about 25% of rows** (135,080), and 1,454 rows with no `Description`
- **Duplicate rows** (5,268 of them)
- **Cancellations and returns**: 9,288 rows have an invoice number starting with `C`, and 10,624 rows have a negative `Quantity`
- **Zero or negative unit prices** (2,517 rows). The minimum price is -11,062 and the minimum quantity is -80,995, which are clearly adjustments and not sales.
- **Non-product stock codes** like `POST`, `DOT`, `M`, `C2` and `BANK CHARGES`, plus gift vouchers. These are fees and vouchers, not items.
- **Manual adjustment invoices** starting with `A` (bad debt write-offs)
- **Inconsistent country naming** (`EIRE` instead of `Ireland`)

### What I did

The notebook works through it in six steps, with a verification printout after each one so I could check that every step did what I intended.

1. **Removed exact duplicates.** 5,268 identical rows were dropped and I confirmed none remained.
2. **Handled missing values.** Missing `CustomerID`s became `-1` as a placeholder and the column was converted to a nullable integer. I kept those rows because they're real sales, just without a customer ID. The 1,454 rows with no `Description` were dropped. *(I originally called these rows "guest customers", but nothing in the data proves they were guest checkouts, so the analysis later treats `-1` as "unknown customer".)*
3. **Split out returns.** 9,725 cancellations and negative-quantity rows went into their own returns table instead of being deleted. Returns are useful for their own analysis, but they would distort revenue if left in the main data.
4. **Removed `UnitPrice <= 0`.** I looked at sample rows first (free items, "amazon" entries and similar) to confirm they weren't real purchases. 584 rows were removed.
5. **Kept only genuine product codes.** Real products have a `StockCode` that starts with five digits (like `85123A`), or belong to the `DCGS` product line (party bags, dog collars and so on). I first tried a hand-written list of codes to remove, but it missed things like a lowercase `m` for "Manual". Keeping only what matches the product pattern turned out to be safer. I checked what the filter removed before trusting it, which is how I caught the `DCGS` items being dropped by mistake. This step removed 2,341 rows. I also renamed `EIRE` to `Ireland`.
6. **Added features and exported.** I created `TotalPrice` (quantity × price) and date parts (`Year`, `Month`, `Day`, `DayOfWeek`, `Hour`), then saved both tables as CSVs.

### Result

| Stage | Rows |
|---|---|
| Raw data | 541,909 |
| After removing duplicates | 536,641 |
| After dropping missing descriptions | 535,187 |
| After splitting out returns | 525,462 |
| After removing zero or negative prices | 524,878 |
| **Final cleaned sales** (`online_retail_cleaned.csv`) | **522,537** |
| **Returns** (`online_retail_returns.csv`) | **9,725** |

The numbers add up: 522,537 sales + 9,725 returns + 9,647 rows removed along the way = 541,909.

The cleaned sales file has 14 columns (the 8 originals plus `TotalPrice` and the five date parts) and about **£10.25 million** in total revenue. That figure still includes the three bulk orders described below, so the analysis works with £9.96 million.

## Stage 2: Analysis

### Step 1: Can I trust the cleaned data?

Before calculating any KPI, I audited the cleaned files. The row counts reconciled, but I found five things that would have quietly distorted the analysis:

1. **`CustomerID = -1` is not a confirmed guest checkout.** It's converted to missing (`NA`), so customer analysis covers identified customers only (about 85% of revenue).
2. **Three invoices (581483, 541431, 556444) were cancelled within minutes** by matching cancellation invoices (same customer, same amount). Together they're worth £284,623, or 2.8% of revenue. I flagged them (`IsReversedOrder`) and left them out of the analysis. This is an assumption: they look like data-entry corrections, not customer returns. I didn't delete anything; the flag can be removed with one line.
3. **The returns file mixed three different things.** I split it into product cancellations (8,668 rows), non-product cancellations such as fees, manual entries and postage (583 rows), and stock adjustments with zero price and no customer (474 rows). Only product cancellations are treated as returns.
4. **`EIRE` was still `EIRE` in the returns file**, so I standardized it to `Ireland` there too.
5. **December 2011 is a partial month** (the data ends on 9 December), so it's never compared with December 2010.

### Step 2: The business questions

| # | Question | Short answer | Where to look |
|---|---|---|---|
| 1 | How do sales change over time? | Flat from January to August, then a sharp rise: £1.03M in September, £1.10M in October, £1.45M in November. Revenue = orders × average order value, and the rise came from orders, not order size. | EDA notebook, 2.1 |
| 2 | When do orders arrive? | 78% between 10:00 and 15:59. Thursday is busiest (21% of orders), Sunday quietest (11%, with orders roughly a quarter to a third smaller). Nothing recorded on Saturdays. | 2.2 |
| 3 | Which products matter? | 249 products (6%) make half the revenue and 844 (22%) make 80%. The top 10 add up to 8%. Unit volume is misleading: only 5 of the top 10 products by units are also in the top 10 by revenue. | 3.1 |
| 4 | Which countries matter? | UK: 85% of revenue, 90% of orders and customers. Among export markets, revenue and customer count tell different stories (see finding 5 above). | 3.2 |
| 5 | How concentrated is customer value? | Top 1% of customers: 30% of identified revenue. Top 10%: 60%. The top 10% order far more often (median 12 orders vs 2) and somewhat more per order. | 4.1 |
| 6 | New or returning customers? | 89% of the August-to-November revenue rise came from returning customers. About 35% of customers bought only once, but they account for just 6.5% of revenue. | 4.2 |
| 7 | How big are cancellations? | About 2.3% of product revenue by value (2.0% leaving out 16 unusually large lines). Cancellation invoices equal 17% of the number of sales invoices, so they're frequent but usually small. | 4.3 |

A few of those answers needed extra care, and I'm happy to explain how I handled them:

- **Customer counts per country** are a floor, because rows without a `CustomerID` can't be counted. I checked how much revenue that affects: none for the Netherlands and Australia, about 5% for Ireland.
- **The "35% bought once" figure is an upper bound.** Customers who first appeared late in the data simply had less time to come back. Among customers who started before April 2011, only 19% bought once.
- **The products with the highest cancellation rates aren't "bad products".** Sixteen very large cancellation lines carry 24% of all cancelled units, and for most of them the same customer had a sale of exactly the same product and quantity within about three days. They look like corrected orders, not unhappy customers.

### How the numbers are defined

| Term | Definition |
|---|---|
| Revenue | Sum of `Quantity × UnitPrice` over genuine product lines. Excludes postage, fees, manual entries, zero-price lines and the three cancelled bulk orders. It's sales, not profit. |
| Order | One distinct `InvoiceNo` |
| Average order value | Revenue ÷ number of orders |
| Identified customer | A distinct `CustomerID` (rows without one are left out of customer analysis) |
| New / returning customer | New in the month of a customer's first purchase in this data, returning afterwards |
| Cancellation rate (value) | Value of product cancellations ÷ product sales |

## Limitations and assumptions

None of these break the analysis, but I'd rather you hear them from me than discover them later. Here's what to keep in mind before quoting a number from this project.

**About the data**

- **It's one year from one retailer.** The autumn peak could be a yearly pattern or a one-off, and I can't tell which. December 2011 covers only nine days.
- **About 15% of revenue has no customer ID.** Everything about customers covers identified customers only. I first called these rows "guest customers", but the data doesn't prove that, so I don't.
- **"New customer" really means "first seen in this data".** Someone who bought in 2010 looks new in early 2011, so the early-2011 new-customer numbers are inflated, and the apparent decline in new customers up to the summer is partly an artefact. The September-October rise is the part I trust.
- **There are no transactions on Saturdays.** Maybe the business was closed, maybe nothing was recorded. The data can't say.
- **The time zone isn't documented.** Hourly patterns are the times as recorded (likely UK time, but unconfirmed).

**About my decisions**

- **Duplicates were removed on an assumption.** I treated 5,268 identical rows as logging errors. Occasionally they could be genuine repeat scans, but I couldn't prove that either way, and the effect should be small.
- **The three bulk orders are excluded on an assumption too.** They look like data-entry corrections because each was cancelled within minutes. If they were real, the cancellation rate would be about 4.6% instead of 2.3%. Both numbers are reported.
- **Sixteen very large cancellation lines are flagged but kept in the main figures.** Where it matters, I show the result with and without them.
- **Cancellations are not matched to their original orders.** So I can't say what share of orders were cancelled, and a cancellation doesn't prove goods were physically returned. Matching them is a project of its own, and I decided it wasn't worth it for this scope.
- **Product concentration is probably understated.** Colour variants of the same item have separate stock codes and are counted as separate products. The product-code filter is also pattern-based, so it could have removed an odd real product (the regex in Step 5 is easy to adjust).
- **Revenue means product sales.** Postage, fees and manual lines are left out.

**About the conclusions**

- **Revenue is not profit.** There's no cost data, so "high-value" just means "high revenue".
- **A few export markets are really a few accounts.** Anything I say about the Netherlands, Ireland or Australia describes one to nine customers.
- **This is description, not explanation.** I can say when customers buy and who drove the growth, but not why. There's no forecasting here, and no claim that anything *caused* anything else.
- **The data doesn't say which customers are wholesalers.** The dataset's documentation says many of them are, which fits the huge order quantities, but I can't tell which accounts they are.

## Run it yourself

1. Clone the repo and install the dependencies:
   ```bash
   pip install -r requirements.txt
   ```
2. Download `Online Retail.xlsx` from the [UCI page](https://archive.ics.uci.edu/dataset/352/online+retail) and put it in `data/raw/`. I don't commit the data itself, since it's large and anyone can download it from the source.
3. Open `notebooks/data_cleaning.ipynb` in Jupyter or VS Code and run all cells. The cleaned files will appear in `data/processed/`.
4. Then open `notebooks/eda_notebook.ipynb` and run all cells. It reads the two CSVs from step 3, so **the order matters**.

I used Python 3.12. The notebooks have f-strings with nested quotes, which need 3.12 or newer. If something looks off partway through the EDA notebook, use **Restart** and then **Run All**: some cells change a table in place, so running them twice in a row can cause errors.

## Project structure

```
.
├── data/
│   ├── raw/          # put Online Retail.xlsx here (not tracked by git)
│   └── processed/    # cleaned CSVs are written here (not tracked by git)
├── notebooks/
│   ├── data_cleaning.ipynb      # Stage 1: raw data to clean sales and returns
│   ├── eda_notebook.ipynb       # Stage 2: audit, analysis, executive summary
│   └── returns_notebook.ipynb   # early look at returns (kept for reference)
├── .gitignore
├── LICENSE
├── requirements.txt
└── README.md
```

## What's next

I deliberately stopped at seven questions rather than adding everything I could think of. If I came back to this, I'd look at:

- **Linking each cancellation to the order it cancelled**, so cancellation rates could be measured per order.
- **Customer segmentation** (for example RFM), now that the unidentified rows and the bulk orders are handled properly.
- **A second year of data**, to find out whether the autumn peak is a real seasonal pattern.

## Credits and license

> Dataset citation: Chen, D. (2015). *Online Retail* [Dataset]. UCI Machine Learning Repository. https://doi.org/10.24432/C5BW33

The dataset is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). See the `LICENSE` file for the license of this repository's code.
