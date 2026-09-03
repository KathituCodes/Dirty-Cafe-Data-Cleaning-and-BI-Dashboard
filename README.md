# Cafe Sales: End-to-End Data Wrangling, EDA, and Dashboarding

A systematic, documented data cleaning and analysis pipeline applied to a 10,000-row intentionally corrupted cafe sales dataset. The project demonstrates multi-layered handling of real-world data quality problems: type corruption, placeholder errors, missing value imputation via unit-economics logic and distribution-preserving sampling, outlier validation, and an interactive Power BI dashboard built on the cleaned output.

---

## Live Demo

Streamlit EDA app: https://dirty-cafe-data-cleaning-and-eda-with-app-dashboard-pidwy9biqm.streamlit.app/

---

## Dashboard

[Cafe Sales Overview dashboard](Doc/Cafe sales overview.PNG)

Interactive Power BI report built on the cleaned dataset: four KPI cards (Total Revenue, Avg Order Value, Total Transactions, Total Units Sold), cross-filtering Location and Payment Method slicers, and four charts (revenue by item, by day of week, and by month). Full build details, DAX measures, and layout notes are in [`Doc/Cafe_Sales_Project_Documentation.docx`](Doc/Cafe_Sales_Project_Documentation.docx).

---

## Project Objectives

- Transform a raw, "dirty" dataset riddled with `ERROR` and `UNKNOWN` placeholders, wrong data types, and structural gaps into a fully analysis-ready table.
- Apply principled, documented decisions at every cleaning step rather than brute-force dropping or mode-filling.
- Derive business insights from the cleaned data on revenue drivers, payment preferences, location behavior, and sales trends, stated at the confidence level the data actually supports.

---

## Dataset

| Property | Detail |
|---|---|
| Source | Synthetic dirty cafe sales data |
| Raw shape | 10,000 rows x 8 columns |
| Cleaned shape | 9,469 rows x 8 columns (531 rows dropped as unrecoverable) |
| Columns | Transaction ID, Item, Quantity, Price Per Unit, Total Spent, Payment Method, Location, Transaction Date |
| Known issues | `ERROR` / `UNKNOWN` text in numerical fields, all columns loaded as `object`, missing values across every column (up to 39.6% in one column), financial inconsistencies |

---

## Cleaning Pipeline

### 1. Initial Audit
- Shape inspection, `df.info()`, and unique value enumeration across all object columns.
- Confirmed all columns loaded as `object` due to string placeholders in numeric fields.
- Generated automated profiling reports (before and after) using `ydata-profiling`.

### 2. Placeholder Standardization
- Replaced `ERROR` and `UNKNOWN` with `np.nan` across the entire dataframe for uniform null handling.

### 3. Data Type Conversion
- Converted `Quantity`, `Price Per Unit`, and `Total Spent` from object to numeric using `pd.to_numeric(errors='coerce')`.
- Converted `Transaction Date` to datetime using `pd.to_datetime(errors='coerce')`.
- `errors='coerce'` was chosen over strict parsing so unparseable values become NaN instead of halting the pipeline.

### 4. Missing Value Imputation

| Column | Strategy | Rationale |
|---|---|---|
| `Price Per Unit` | Mode by Item group | Items have fixed menu prices; mode recovery is deterministic |
| `Item` | Reverse price-to-item map (unique-price-only) | Only applied where a price maps to exactly one item, to avoid ambiguous assignment |
| `Total Spent` | `Quantity * Price Per Unit` | Unit-economics recovery preserves financial integrity |
| `Quantity` | `Total Spent / Price Per Unit` | Inverse recovery from a verified total |
| `Payment Method` | Random distribution-proportional sampling | Preserves the real observed split; avoids inflating the modal category |
| `Location` | Random distribution-proportional sampling | Same reasoning as Payment Method |
| `Item` (residual) | Random distribution-proportional sampling | Same reasoning as above, applied after the price-based recovery step |
| `Quantity`, `Price Per Unit`, `Total Spent`, `Transaction Date` (residual) | Drop rows | Filling numeric columns with a string placeholder reverts them to text and breaks all downstream math; affected volume was ~0.7% for numeric fields, 4.6% for dates |

### 5. Outlier Detection and Handling
- Visualized distributions using boxplots.
- Applied the IQR method across `Quantity`, `Price Per Unit`, and `Total Spent`; 259 rows flagged in `Total Spent` (2.7% of cleaned data).
- Validated flagged rows against business domain logic: the maximum physically possible transaction is 5 units x $5.00 = $25.00. Every flagged row topped out at exactly $25.00.
- Decision: **retained** all flagged outliers, they are legitimate high-value orders, not corrupted data. No capping applied.

### 6. Duplicate Check
- Confirmed zero duplicate rows in the cleaned dataset.

---

## Exploratory Data Analysis

| Analysis | Finding |
|---|---|
| Revenue by item | Salad, Sandwich, and Smoothie lead by total revenue; Cookie is lowest. Transaction *counts* per item are far more even (1,102–1,275 across all 8 items), so revenue ranking is driven by unit price and quantity mix, not one item being ordered disproportionately more often |
| Transaction value distribution | 57% of transactions fall between $3 and $10 |
| Daily/monthly sales trend | Flat across all 12 months and all 7 days of the week, no growth trend, seasonality, or weekday effect detected |
| Payment method share | Near-even three-way split: Credit Card 33.5%, Digital Wallet 33.5%, Cash 32.9%. No method meaningfully outperforms another |
| Location share | Near-even split: In-store 50.6%, Takeaway 49.4%. Neither channel dominates |
| Revenue by item and payment method | Proportional across payment types, consistent with the near-even overall split above |

**Note on interpretation:** Payment Method, Location, and residual Item values were partly filled using distribution-preserving random imputation (covering roughly 32–40% of two of those columns). That method was deliberately chosen to avoid distorting the true split, so the near-even results above are the expected outcome of that choice, not necessarily proof of an underlying real-world 50/50 customer behavior. Combined with the flat time trend and the near-uniform item transaction counts, the overall pattern is consistent with data generated to be statistically uniform rather than sampled from an operating cafe. Read the business recommendations below with that in mind.

---

## Business Recommendations

1. **Fix upstream data collection first.** The volume of `ERROR` and `UNKNOWN` placeholders, and the fact that Payment Method and Location were missing in 32–40% of rows, points to a Point-of-Sale data entry gap at the source. This is the highest-confidence, most actionable finding in the project, independent of any of the imputed splits above.
2. **Don't optimize for one payment channel or location yet.** With Credit Card, Digital Wallet, and Cash essentially tied, and In-store/Takeaway close to a 50/50 split, there isn't a clear channel to prioritize based on this dataset. A real, non-imputed sample would be needed before committing resources to one channel over another.
3. **The flat sales trend is worth investigating operationally, not just imputing around.** Whether the flatness reflects a real demand ceiling or is itself an artifact of how this dataset was generated, either explanation is worth confirming against a live POS feed before drawing further business conclusions from it.

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Python 3 | Core language |
| pandas | Data manipulation and cleaning |
| numpy | Numerical operations and NaN handling |
| matplotlib / seaborn | Static visualizations |
| plotly | Interactive sales trend chart |
| ydata-profiling | Automated before/after profiling reports |
| Power BI | Interactive dashboard on the cleaned dataset |

---

## Repository Structure

```
cafe-sales-data-cleaning-eda/
|
|-- Data/
|   |-- dirty_cafe_sales_csv.xls        # Raw input data
|   |-- Clean_Cafe_Sales_Data.csv       # Cleaned output data
|
|-- Doc/
|   |-- Cafe_Sales_Project_Documentation.docx   # Full write-up: cleaning rationale, EDA, dashboard build
|   |-- Cafe_sales_overview.PNG                 # Dashboard screenshot
|
|-- Dirty_Cafe_Sales_Data_cleaning.ipynb        # Full cleaning + EDA notebook
|-- Cafe_Sales_Dashboard.pbix                   # Power BI report file
|-- README.md
```

---

## How to Run

1. Clone the repository.
2. Install dependencies:
   ```bash
   pip install pandas numpy matplotlib seaborn plotly ydata-profiling
   ```
3. Open `Dirty_Cafe_Sales_Data_cleaning.ipynb` in Jupyter or Google Colab.
4. Run all cells sequentially. The notebook is structured to be executed top to bottom with outputs displayed at each stage.
5. To explore the dashboard, open `Cafe_Sales_Dashboard.pbix` in Power BI Desktop (free), or view the screenshot above for a static preview.

---

## Author

**Urbanus Kathitu** | [github.com/KathituCodes](https://github.com/KathituCodes)
