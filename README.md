
<img width="310" height="311" alt="analysis" src="https://github.com/user-attachments/assets/8287b5f4-5a77-47bf-825e-1b682cf0e1cc" />
# AtliQo Bank Credit Card Launch — Customer Segmentation & A/B Testing

End-to-end data science project that identifies an untapped customer segment for a new credit card launch, then validates the launch decision with a statistically rigorous A/B test.

<img width="310" height="311" alt="analysis" src="https://github.com/user-attachments/assets/8287b5f4-5a77-47bf-825e-1b682cf0e1cc" />
---

## 📌 Business Problem

AtliQo Bank wants to launch a new credit card but does not want to market it to its entire customer base blindly. The bank needs to know:

1. **Who** is the most promising, currently underserved customer segment?
2. **Will** a new card actually change spending behaviour for that segment, or is the expected lift just noise?

This project answers both questions using real customer, credit, and transaction data, followed by a controlled pilot campaign.

## 🗂️ Project Structure

```
atliqo-bank-credit-card-launch/
│
├── phase_1_atliqo_bank.ipynb        # EDA, data cleaning, segmentation
├── phase_2_atliqo_bank.ipynb        # Power analysis, A/B test, hypothesis testing
├── datasets/
│   ├── customers.csv
│   ├── credit_profiles.csv
│   └── transactions.csv
├── data/
│   └── avg_transactions_after_campaign.csv
└── README.md
```

## 🔧 Tech Stack

| Category | Tools |
|---|---|
| Language | Python 3 |
| Data wrangling | Pandas, NumPy |
| Visualization | Matplotlib, Seaborn |
| Statistics | SciPy, Statsmodels |
| Data sources | CSV, MySQL (`mysql.connector`) |
| Environment | Jupyter Notebook |

## 🧭 Project Workflow

```
Raw Data (Customers, Credit Profiles, Transactions)
        │
        ▼
Data Cleaning (nulls, duplicates, business-rule outliers)
        │
        ▼
Exploratory Data Analysis (univariate → bivariate → correlation)
        │
        ▼
Target Segment Identification  →  Customers aged 18–25
        │
        ▼
Power Analysis  →  Sample size for A/B test
        │
        ▼
Pilot Campaign (Test vs Control groups, 2 months)
        │
        ▼
Two-Sample Z-Test on average transaction amount
        │
        ▼
Statistically validated go/no-go recommendation
```

---

## Phase 1 — Exploratory Data Analysis & Customer Segmentation

**Datasets:** `customers` (1,000 rows), `credit_profiles` (1,004 rows), `transactions` (500,000 rows).

### Data Cleaning (business-rule driven, not just statistical)

| Column | Issue | Treatment |
|---|---|---|
| `annual_income` | 50 nulls | Filled with **occupation-wise median** income |
| `annual_income` | Implausible values (< $100) | Replaced with occupation-wise median |
| `age` | Outliers (age < 15 or > 80, max = 135) | Replaced with **occupation-wise median age** |
| `credit_profiles` | 4 duplicate `cust_id` rows | Dropped, keeping the most recent record |
| `credit_limit` | Nulls | Filled using the **mode of credit_limit within each credit-score band** (data-driven relationship, not a flat average) |
| `outstanding_debt` | Values exceeding `credit_limit` (impossible in practice) | Capped at `credit_limit` |
| `platform` (transactions) | Nulls | Filled with **mode** (Amazon — the dominant platform) |
| `tran_amount` | Zero-value transactions in a specific platform/category/payment combo | Replaced with the **segment median** |
| `tran_amount` | Upper-tail outliers (IQR rule) | Replaced with **category-wise mean** |

Every treatment decision was deliberately tied back to business logic (e.g. "debt cannot exceed credit limit") rather than blindly applying statistical rules — a key differentiator of this analysis.

### Key EDA Insights

- Annual income is **right-skewed**; income varies meaningfully by occupation, gender, location, and marital status.
- Customers split into three age bands: **18–25 (24.6%)**, **26–48 (56.7%)**, **49–65 (18.7%)**.
- **`credit_score` and `credit_limit` are strongly correlated (r ≈ 0.85)** — banks scale credit limits in tiers as credit score rises.
- `credit_limit` also correlates with `annual_income` (r ≈ 0.58) and `outstanding_debt` (r ≈ 0.81).
- Credit card usage as a payment method is **noticeably lower in the 18–25 age group** than in older groups.
- Top spending categories for 18–25-year-olds: **Electronics, Fashion & Apparel, Beauty & Personal Care**, mainly via **Amazon, Flipkart, Alibaba**.

### 🎯 Target Segment Decision

**Customers aged 18–25** were selected as the launch target because they are:
- A sizeable group (~25% of the customer base) that is **currently underserved**.
- Earning **below $50K** on average, with thin credit history.
- Showing **low credit card adoption**, i.e. high headroom for a new product to change behaviour.
- Already spending in categories well suited to card-linked rewards (electronics, fashion, beauty).

---

## Phase 2 — A/B Test Design & Hypothesis Testing

### Step 1: Power Analysis (sample size)

Using `statsmodels.stats.power.tt_ind_solve_power` with `alpha = 0.05`, `power = 0.8`:

| Effect Size | Required Sample Size (per group) |
|---|---|
| 0.1 | 1,570 |
| 0.2 | 393 |
| 0.3 | 175 |
| **0.4** | **~99–100** |
| 0.5 | 63 |
| 1.0 | 16 |

An effect size of **0.4** was agreed with the business as the minimum meaningful lift worth detecting, balancing statistical rigor with the campaign budget — giving a required sample of ~100 customers per group.

### Step 2: Campaign Design

- Pool of **~246 eligible customers** aged 18–25.
- **100 customers** selected for the trial launch (per the power analysis).
- Campaign ran for **2 months** (10 Sep 2023 – 10 Nov 2023).
- **~40% conversion** — 40 of the 100 test customers actually started using the new card → this defined the **test group (n=40)**.
- A matching **control group of 40 customers** (mutually exclusive of the test pool) continued using existing payment methods.
- Daily average transaction amounts were tracked for both groups across the campaign window (62 observations per group).

### Step 3: Hypothesis Testing — Two-Sample Z-Test

**H₀:** New card has no effect → mean(test) ≤ mean(control)
**H₁:** New card increases average transaction amount → mean(test) > mean(control)  *(one-tailed, right-tailed test)*

| Metric | Control Group | Test Group |
|---|---|---|
| Mean avg. transaction | $221.18 | **$235.98** |
| Std. dev | 21.36 | 36.66 |
| n | 62 | 62 |

**Results:**

| Method | Z-statistic | p-value | Critical Z (α=0.05) |
|---|---|---|---|
| Manual Z-test | 2.747 | 0.00301 | 1.645 |
| `statsmodels.stats.weightstats.ztest` | 2.748 | 0.00300 | 1.645 |

Both approaches agree: **Z > critical Z** and **p-value < 0.05** → **reject the null hypothesis.**

### ✅ Conclusion

The new credit card produced a **statistically significant ~6.7% increase** in average transaction amount among 18–25-year-old customers (from $221.18 to $235.98). The result is not attributable to chance (p = 0.003), supporting a **full-scale rollout** of the card to this segment.

---

## 💡 How to Read / Re-use This Analysis

1. Start with `phase_1_atliqo_bank.ipynb` top-to-bottom — every cleaning decision has a markdown explanation directly above the code that performs it.
2. Pay attention to the **"why" behind each cleaning choice** — most outlier/null treatments here are driven by business rules (e.g. *debt can't exceed credit limit*), not blanket statistical formulas. This is the main analytical skill the project demonstrates.
3. The correlation heatmap and age-group breakdown cells are the fastest way to re-derive the segmentation logic if the underlying data changes.
4. `phase_2_atliqo_bank.ipynb` is self-contained for the experiment design — change `effect_size`, `alpha`, or `power` at the top to see how the required sample size shifts.
5. The Z-test is implemented two ways (manual formula and `statsmodels`) deliberately, as a built-in check that the manual statistics are correct.

## 🚀 How to Run

```bash
git clone <your-repo-url>
cd atliqo-bank-credit-card-launch
pip install pandas numpy seaborn matplotlib scipy statsmodels mysql-connector-python
jupyter notebook
```

Place `customers.csv`, `credit_profiles.csv`, `transactions.csv` in `datasets/`, and `avg_transactions_after_campaign.csv` in `data/` (or connect to a MySQL instance — both import options are included in the notebook).

## 📈 Possible Extensions

- Segment-level A/B testing across other age bands or occupations.
- Survival/churn analysis on card adoption over a longer post-campaign window.
- A logistic regression model to predict conversion probability per customer, enabling smarter targeting before the next campaign.

## 👤 Author

**Bhanu Prakash Sige**
MSc Data Science, University of Roehampton, London
