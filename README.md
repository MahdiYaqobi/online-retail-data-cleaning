# Online Retail: Data Cleaning

I took a messy, real-world transactions dataset and turned it into something I'd actually trust for analysis. This repo is the cleaning part of that work: what was wrong with the data, what I did about it, and why.

## About the data

The [Online Retail dataset](https://archive.ics.uci.edu/dataset/352/online+retail) comes from the UCI Machine Learning Repository. It contains every transaction made between **1 December 2010 and 9 December 2011** by a UK-based online store that sells all-occasion gifts. Many of its customers are wholesalers, which explains some of the very large order quantities you'll see.

It has **541,909 rows** and 8 columns:

| Column | What it is |
|---|---|
| `InvoiceNo` | Invoice number. A leading `C` means the order was cancelled. |
| `StockCode` | Product code |
| `Description` | Product name |
| `Quantity` | Units per transaction |
| `InvoiceDate` | Date and time of the transaction |
| `UnitPrice` | Price per unit (in GBP) |
| `CustomerID` | Customer identifier |
| `Country` | Customer's country |

> Citation: Chen, D. (2015). *Online Retail* [Dataset]. UCI Machine Learning Repository. https://doi.org/10.24432/C5BW33

## What was wrong with it

Before changing anything, I profiled the data. Here is what stood out:

- **Missing `CustomerID` on about 25% of rows** (135,080), which are guest checkouts, and 1,454 rows with no `Description`
- **Duplicate rows** from logging errors (5,268 of them)
- **Cancellations and returns**: 9,288 invoices start with `C`, and 10,624 rows have a negative `Quantity`
- **Zero or negative unit prices** (2,517 rows). The minimum price is -11,062 and the minimum quantity is -80,995, which are clearly adjustments and not sales.
- **Non-product stock codes** like `POST`, `DOT`, `M`, `C2` and `BANK CHARGES`, plus gift vouchers. These are fees and vouchers, not items.
- **Manual adjustment invoices** starting with `A` (bad debt write-offs)
- **Inconsistent country naming** (`EIRE` instead of `Ireland`)

## What I did

The notebook works through it in six steps, with a verification printout after each one so I could check that every step did what I intended.

1. **Removed exact duplicates.** 5,268 identical rows were dropped and I confirmed none remained.
2. **Handled missing values.** Missing `CustomerID`s became `-1` (guest customer) and the column was converted to a nullable integer. I kept those rows because they're real sales, just anonymous. The 1,454 rows with no `Description` were dropped.
3. **Split out returns.** 9,725 cancellations and negative-quantity rows went into their own `df_returns` table instead of being deleted. Returns are useful for their own analysis, but they would distort revenue if left in the main data.
4. **Removed `UnitPrice <= 0`.** I looked at sample rows first (free items, "amazon" entries and similar) to confirm they weren't real purchases. 584 rows were removed.
5. **Kept only genuine product codes.** Real products have a `StockCode` that starts with five digits (like `85123A`), or belong to the `DCGS` product line (party bags, dog collars and so on). I first tried a hand-written list of codes to remove, but it missed things like a lowercase `m` for "Manual". Keeping only what matches the product pattern turned out to be safer. I checked what the filter removed before trusting it, which is how I caught the `DCGS` items being dropped by mistake. This step removed 2,341 rows. I also renamed `EIRE` to `Ireland`.
6. **Added features and exported.** I created `TotalPrice` (quantity × price) and date parts (`Year`, `Month`, `Day`, `DayOfWeek`, `Hour`), then saved both tables as CSVs.

## Result

| Stage | Rows |
|---|---|
| Raw data | 541,909 |
| After removing duplicates | 536,641 |
| After dropping missing descriptions | 535,187 |
| After splitting out returns | 525,462 |
| After removing zero or negative prices | 524,878 |
| **Final cleaned sales** (`online_retail_cleaned.csv`) | **522,537** |
| **Returns** (`online_retail_returns.csv`) | **9,725** |

The cleaned file has 14 columns (the 8 originals plus `TotalPrice` and the five date parts). Total revenue in the cleaned sales data comes to about **£10.25 million**.

## Things to keep in mind

- **Cancelled orders aren't matched to their originals.** Returns are separated out, but I didn't link each cancellation back to the invoice it cancelled. So an order that was later cancelled still sits in the sales data, which can inflate revenue and quantities, especially for a few huge orders. If you do revenue analysis, check the largest quantities first.
- `df_returns` includes every row with a negative quantity, not only invoices starting with `C`. A few of those are stock adjustments rather than customer returns.
- Guest customers all share `CustomerID = -1` (133,583 rows after the earlier steps), so exclude them before doing any per-customer analysis (RFM, churn, lifetime value).
- The product-code rule is based on patterns I found by inspecting the data. If you find a real product it wrongly removed, the regex in Step 5 is easy to adjust.

## Run it yourself

1. Clone the repo and install the dependencies:
   ```bash
   pip install -r requirements.txt
   ```
2. Download `Online Retail.xlsx` from the [UCI page](https://archive.ics.uci.edu/dataset/352/online+retail) and put it in `data/raw/`. I don't commit the data itself, since it's large and anyone can download it from the source.
3. Open `notebooks/data_cleaning.ipynb` in Jupyter or VS Code and run all cells. The cleaned files will appear in `data/processed/`.

I used Python 3.12. The notebook has f-strings with nested quotes, which need 3.12 or newer.

## Project structure

```
.
├── data/
│   ├── raw/          # put Online Retail.xlsx here (not tracked by git)
│   └── processed/    # cleaned CSVs are written here (not tracked by git)
├── notebooks/
│   └── data_cleaning.ipynb
├── .gitignore
├── requirements.txt
└── README.md
```

## What's next

The cleaned data is ready for exploratory analysis, customer segmentation and sales trend work. That will go in separate notebooks.
