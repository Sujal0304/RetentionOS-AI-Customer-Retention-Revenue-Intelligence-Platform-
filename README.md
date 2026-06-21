# RetentionOS-AI-Customer-Retention-Revenue-Intelligence-Platform-

A customer retention tool that doesn't just predict who will churn — it answers the question a real Customer Success team actually has: **given limited time to make calls this week, who should we call, and what should we say?**

Built on the IBM Telco Customer Churn dataset (7,043 customers), this project reframes churn prediction as a budget-constrained optimization problem, then uses a small, tightly-scoped GenAI layer to translate the numbers into plain-language recommendations a non-technical manager can act on immediately.

---

## Why this exists

Most churn projects stop at "here's a model that predicts who will leave." That's not actually useful on its own — a Customer Success team can't call every at-risk customer, so the real decision isn't *who is at risk*, it's *who is worth acting on first, given limited capacity*.

This project ranks customers by **Expected Value at Risk** (churn probability × customer lifetime value), not churn risk alone, so a high-risk-but-low-value customer doesn't crowd out a moderate-risk-but-high-value one.

---

## Pipeline overview
### 1. Data Cleaning
- Fixed 11 blank `Total Charges` rows (new customers with zero billing history)
- Validated no duplicate customer IDs, no out-of-range values

### 2. EDA / Visualization
Three findings that directly shaped later design decisions, not decorative charts:
- **Tenure vs. churn is non-linear** — churn drops sharply in the first year, then flattens. This is why the Health Score uses a log-scaled tenure component instead of a linear one.
- **Payment method matters even within the same contract type** — electronic check customers churn meaningfully more than mailed check customers, even on identical contracts.
- **Fiber optic customers churn more despite paying a premium** — confirms a real "paying more, satisfied less" pattern in the data.

### 3. Preprocessing
A single `scikit-learn` `Pipeline` with a `ColumnTransformer` handles scaling and one-hot encoding. This avoids data leakage during cross-validation, since preprocessing is fit only on each fold's training data, not the full dataset at once.

**Explicitly excluded from model inputs:** IBM's pre-computed `Churn Score`, `CLTV`, and `Churn Reason` columns. These are derived *after* churn happened, so using them as predictive features would be leakage — the model would just be learning to copy a label, not predict an outcome.

### 4. Model Building and Validation
- Compared **Logistic Regression** against **Gradient Boosting** using 5-fold stratified cross-validation (not a single train/test split)
- Ran a **paired t-test** on the fold-level AUC scores to check whether the difference between models was statistically real, or just noise (`p = 0.28` — not significant)
- **Chose Logistic Regression** for production, since it gave up no meaningful accuracy while staying fully interpretable — every coefficient can be explained directly, which matters because its output feeds a transparent, auditable Health Score downstream

### 5. Health Score (0–100)
A weighted, fully transparent formula — not a second black-box model:

| Component | Weight | Why |
|---|---|---|
| Churn probability (inverted) | 40% | Core risk signal from the model |
| Tenure (log-scaled) | 25% | Matches the real non-linear churn-vs-tenure curve found in EDA |
| CLTV (normalized) | 20% | Customers worth more should weigh more in their own score |
| Contract stability | 15% | Month-to-month vs. annual vs. biennial |

### 6. Budget Allocator
Ranks **active** customers (you can't retain someone who already left) by Expected Value at Risk, and returns the top N a team actually has capacity to act on. Tested across budget sizes (20 → 500) to show **diminishing returns** — average value captured per customer drops as the budget grows, proving there's a real, data-backed optimal stopping point for retention spend, not an assumed one.

### 7. GenAI Explanation Layer
For each customer in the allocated budget, an LLM (Llama 3.3 70B via the Groq API) is given a small set of real, pre-computed numbers — Churn Risk, Health Score, Revenue, Revenue at Risk, tenure, contract, internet service, and payment method — and asked to act as an analyst: decide a priority tier (P1/P2/P3), explain its reasoning in plain English, and suggest one specific action.

**Design choices that matter here:**
- The model never invents facts outside what it's given — it reasons over real numbers, not free text
- The prompt explicitly bans technical vocabulary ("probability," "score," "percentage points") since the audience is a non-technical manager, not a data scientist
- A **rule-based fallback** generates a usable (if simpler) note if the API call fails for any reason — rate limits, network issues, bad keys — so the pipeline never breaks even when the AI call does

### 8. Power BI Dashboard
*(In progress / to be added)*

Planned views:
- **Priority queue** — the allocated customer list, sortable by Priority tier and Revenue at Risk
- **Account detail card** — Health Score trend, contract/payment context, and the AI-generated note for a single customer
- **Portfolio view** — churn rate and revenue-at-risk breakdowns by contract type, tenure band, and segment
- **Budget slider** — lets a user adjust how many customers are in scope and see Expected Value captured update live

---

## Tech stack

`Python` `Pandas` `NumPy` `Scikit-learn` `SciPy` `Matplotlib` `Groq API (Llama 3.3 70B)` `Power BI`

---

## Known limitations (stated honestly, not hidden)

- **Expected Value at Risk is an upper bound, not a guaranteed savings figure.** It doesn't account for intervention success rate (e.g., a retention call working only 30% of the time) — a more complete model would multiply by an estimated save probability, which this dataset doesn't provide.
- **The dataset is a single snapshot**, not a time series — there's no way to validate the allocator against what *actually* happened after an intervention, since no historical "we called this customer and here's what happened" data exists.
- **The Health Score formula's weights (40/25/20/15) are a reasoned starting point, not a fitted optimum.** They reflect the relative importance found in EDA, not a tuned result.

---

## Running it

1. Open the notebook in Google Colab
2. Upload `Telco_customer_churn.xlsx` when prompted
3. Get a free API key at [console.groq.com](https://console.groq.com) and add it where indicated in the AI Explanation section
4. Run all cells top to bottom
5. The final cell exports `power_bi_export.csv` for use in the Power BI dashboard
