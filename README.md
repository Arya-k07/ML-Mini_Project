# Banking Product Cross-Sell Recommendation System  
## Updated Mini Project Plan — Multi-Pattern Market Basket Analysis + Flask + Frontend + Email Campaigns

## Project Status & Quick Start

| Item | Status |
|---|---|
| Phase 1 - Dataset generator | ✅ **Done** - `generate_banking_dataset.py` |
| Dataset size | **20,000 customers**, 21 products, **~99,600 customer-product rows** |
| Planted cross-sell rules | All 14 embedded; validated (see *Dataset Generation Logic*) |
| Phases 2-7 | Not started |

```bash
pip install pandas numpy faker        # faker is optional (built-in name pool is used if missing)
python generate_banking_dataset.py    # writes ./data/*.csv and prints a rule-validation table
```

Dataset quality checks from the generator run (seed 42):

| Check | Result |
|---|---|
| Customers | 20,000 |
| Customer-product rows | ~99,600 (avg ~5.0 products per customer) |
| Unique emails | 20,000 (all unique, name-based, 8 domains) |
| Distinct product baskets | ~2,600 (not 8 copy-paste templates) |
| Unique names | ~6,000 with the built-in pool; far more with Faker installed |

## Table of Contents

- [1. Updated Project Idea](#1-updated-project-idea)
- [2. Project Title](#2-project-title)
- [3. Main Objectives](#3-main-objectives)
- [4. Expert Recommendations & Best Practices](#4-expert-recommendations--best-practices)
- [5. System Overview](#5-system-overview)
- [6. Technology Stack](#6-technology-stack)
- [7. Customer Behavior Profiles](#7-customer-behavior-profiles)
- [8. Cross-Sell Patterns to Build](#8-cross-sell-patterns-to-build)
- [9. Dataset Requirements](#9-dataset-requirements)
- [10. Banking Products](#10-banking-products)
- [11. Dataset Generation Logic](#11-dataset-generation-logic)
- [12. Recommendation Engine Logic](#12-recommendation-engine-logic)
- [13. Eligibility Rules](#13-eligibility-rules)
- [14. Email Campaign Feature](#14-email-campaign-feature)
- [15. Database Design](#15-database-design)
- [16. Backend API Endpoints](#16-backend-api-endpoints)
- [17. Frontend Pages](#17-frontend-pages)
- [18. Frontend Route Summary](#18-frontend-route-summary)
- [19. Project Folder Structure](#19-project-folder-structure)
- [20. Development Plan](#20-development-plan)
- [21. Testing Plan](#21-testing-plan)
- [22. Demo Script](#22-demo-script)
- [23. Report Structure](#23-report-structure)
- [24. Future Scope](#24-future-scope)
- [25. Final Deliverables](#25-final-deliverables)

---

## 1. Updated Project Idea

Build a full-stack banking cross-sell recommendation system that discovers **multiple meaningful product-ownership patterns** using Market Basket Analysis.

Instead of generating random product ownership data, the synthetic dataset will contain hidden customer behavior groups:

- Young / Education customers
- Salaried customers
- Investors
- Homeowners
- Vehicle owners
- Retired / senior customers
- High-value wealth customers

The backend will use Apriori association-rule mining to discover relationships such as:

```text
Pension Account + Pension Card -> Pension Scheme
Salary Account + Credit Card -> Personal Loan
Home Loan -> Home Insurance
Savings Account + Fixed Deposit -> Mutual Fund
Mutual Fund + Demat Account -> Wealth Management Service
```

Then the frontend will show customer-specific recommendations and allow the bank employee to send personalized offer emails.

---

## 2. Project Title

**Banking Product Cross-Sell Recommendation System using Multi-Pattern Market Basket Analysis, Flask, and Automated Email Campaigns**

---

## 3. Main Objectives

1. Generate a realistic synthetic banking dataset with multiple customer profiles.
2. Discover frequently held banking-product combinations using Apriori.
3. Generate association rules with support, confidence, and lift.
4. Recommend the next best banking product for each customer.
5. Check eligibility before recommending a product.
6. Send personalized product-offer emails for selected recommendations.
7. Provide an admin dashboard for customers, products, rules, campaigns, and analytics.
8. Demonstrate at least 8–10 meaningful cross-sell patterns.
9. Validate the mining step: compare the 14 planted rules against the rules Apriori actually discovers (rule recovery rate).
10. Evaluate the recommender with a hold-out test (hit rate / precision@k) against a "most popular product" baseline.
11. Make email campaigns safe to demo (TEST_MODE, dry-run, full send log) so no mail reaches generated addresses.

---

## 4. Expert Recommendations & Best Practices

Ordered by how much each one strengthens the project. Items 1-5 are strongly recommended; items 6-8 are planning and polish.

### 4.1 Be upfront that the data is synthetic

Apriori will "discover" the rules the generator planted, so a reviewer may call the result circular. Turn this into a strength:

- Build a **rule-validation step** comparing the 14 planted rules with what Apriori recovers.
- Report a table: planted rule, target confidence, generated confidence, discovered (yes / no), discovered confidence and lift.
- Report the **recovery rate** (rules found / 14) and explain any misses (usually a support or confidence threshold).
- State the synthetic-data limitation explicitly in the report.

Implementation: `rule_validation.ipynb` and `GET /api/analysis/validation`.

### 4.2 Measure the recommender, not just the rules

Use a hide-one hold-out test:

```python
import random

def evaluate(baskets, rules, popularity, k=3, seed=42):
    random.seed(seed)
    hits_rules = hits_pop = total = 0
    for basket in baskets:
        if len(basket) < 3:
            continue
        hidden = random.choice(sorted(basket))
        visible = basket - {hidden}
        recs = recommend(visible, rules, top_k=k)                       # rule-based
        base = [p for p in popularity if p not in visible][:k]          # baseline
        hits_rules += hidden in recs
        hits_pop += hidden in base
        total += 1
    return hits_rules / total, hits_pop / total
```

- Report **hit rate @3** and **precision @3** for the rule engine and for the "most popular products" baseline.
- The system is only convincing if it beats the baseline; show the comparison as a table or bar chart.
- Implementation: `recommender_evaluation.ipynb`, `evaluation_service.py`, `POST /api/analysis/evaluation`.

### 4.3 Adjust thresholds and read lift correctly

About 98% of customers own a Savings Account, so rules built on it (Savings Account + Debit Card -> Credit Card, lift ≈ 1.09) will always have low lift even when confidence is fine. Lift measures how much *more* often the product appears than by chance, and that is nearly impossible for a near-universal item.

| Purpose | min_support | min_confidence | min_lift |
|---|---:|---:|---:|
| Recover all 14 planted rules | 0.03 (0.025 for headroom) | 0.35 | 1.05 |
| Clean "top rules" demo | 0.03 | 0.45 | 1.2 |

Also add an `Income: High` pseudo-item to baskets so rule 13 (High Income + Mutual Fund -> Wealth Management Service) can be discovered. Explain the near-universal-item effect in the report - it shows you understand lift.

### 4.4 Use the new columns in the eligibility check

The dataset now includes `credit_score`, `annual_income` and `avg_monthly_balance`, so eligibility can go beyond age: credit-score cut-offs for credit products, KYC `Verified` for investment products, income threshold for Wealth Management, and inactive accounts excluded from campaigns. Full table in *Eligibility Rules*.

### 4.5 Never e-mail the generated addresses

Generated addresses like `name1234@gmail.com` may belong to real people. Add `TEST_MODE` (default ON) that redirects every message to your own inbox with the intended customer in the subject, add a dry-run option, and log intended vs actual recipient. Never commit `.env`. Details and code in *Email Campaign Feature*.

### 4.6 Build in this order

1. Dataset ✅
2. Apriori + recommendation engine tested from the command line
3. Validation and evaluation
4. Flask APIs
5. Frontend
6. Emails last

A working engine plus one clean dashboard beats a half-finished full UI. A single admin account is enough for login.

### 4.7 Add extra metrics to the report

- Besides support, confidence and lift, include **leverage** and **conviction** (mlxtend's `association_rules` returns both).
- Show a **heatmap** of how often product pairs are owned together.
- Show a **bar chart of rule strength by customer profile**.

### 4.8 Optional extension: rule-based vs machine learning

Train a logistic regression and a decision tree to predict whether a customer owns a given product from age, income, credit score and owned products, then compare AUC / hit rate@3 with the rule-based recommender. This turns the project from one technique into a small comparative study (Phase 8).

### 4.9 Known limitations of the generated dataset

- Planted rules make rule discovery easy (see 4.1).
- All product holdings are `Active`; there are no closed or dormant products.
- Without Faker installed, names repeat (~6,000 unique names across 20,000 customers); install Faker for more variety. E-mails are unique either way.
- Product ownership respects age limits only, not credit score or KYC (those are applied by the eligibility engine).
- Rules 3, 4, 12 and 14 land within ~5 points of their target ranges rather than exactly inside them.

---

## 5. System Overview

```text
Synthetic Banking Dataset
            |
            v
Flask Backend
            |
            +--> SQLite Database
            |
            +--> Market Basket Analysis Engine
            |         |
            |         +--> Customer Baskets
            |         +--> One-Hot Encoding
            |         +--> Apriori Algorithm
            |         +--> Association Rules
            |
            +--> Recommendation Engine
            |         |
            |         +--> Match customer products with rules
            |         +--> Remove already-owned products
            |         +--> Apply eligibility rules
            |         +--> Rank by confidence and lift
            |
            +--> Email Campaign Service (TEST_MODE / dry-run)
            |
            +--> Validation & Evaluation Service
            |         |
            |         +--> Planted vs discovered rules
            |         +--> Hit rate / precision@k vs popularity baseline
            |
            v
Frontend Dashboard
```

---

## 6. Technology Stack

| Layer | Technology |
|---|---|
| Frontend | HTML, CSS, JavaScript, Bootstrap |
| Charts | Chart.js or Plotly |
| Backend | Python Flask |
| Database | SQLite |
| Data processing | pandas, NumPy |
| Association mining | mlxtend |
| Fake customer data | Faker (optional - generator falls back to a built-in name pool) |
| Evaluation / optional ML | scikit-learn, matplotlib / seaborn |
| Email | Flask-Mail or smtplib |
| API testing | Postman / Thunder Client |
| Version control | Git and GitHub |

Install:

```bash
pip install flask flask-sqlalchemy flask-mail python-dotenv pandas numpy faker mlxtend scikit-learn matplotlib seaborn
```

---

## 7. Customer Behavior Profiles

The dataset generator will create customers belonging to hidden behavioral groups. Each group will have a base product set and likely follow-up products.

| Profile | Target share | Base products | Likely cross-sell products |
|---|---:|---|---|
| Young / Education | 15% | Savings Account, Debit Card, Education Loan | Credit Card |
| Salaried | 22% | Salary Account, Savings Account, Debit Card | Credit Card, Personal Loan |
| Investor | 18% | Savings Account, Fixed Deposit, Debit Card | Mutual Fund, Demat Account |
| Homeowner | 12% | Savings Account, Home Loan, Debit Card | Home Insurance, Life Insurance |
| Vehicle Owner | 10% | Savings Account, Vehicle Loan, Debit Card | Vehicle Insurance, Life Insurance |
| Retired / Senior | 13% | Pension Account, Pension Card, Savings Account | Pension Scheme, Health Insurance |
| High-Value Customer | 10% | Savings Account, Fixed Deposit, Mutual Fund, Demat Account | Wealth Management Service |
| General Customer | Remaining | Savings Account, Debit Card | Fixed Deposit, Health Insurance |

The exact percentages can be adjusted, but every profile should have enough customers for Apriori to detect its pattern.

> **Note - shares are normalised.** The target shares above add up to 110% (General Customer is listed as 10%, not "remaining"), so the generator rescales them. Realised shares in the 20,000-customer run: Salaried 20.2%, Investor 16.4%, Young / Education 13.6%, Retired / Senior 11.9%, Homeowner 11.1%, General 9.2%, Vehicle Owner 9.0%, High-Value 8.6%. Every profile still has 1,700+ customers.

---

## 8. Cross-Sell Patterns to Build

The generator should intentionally include these 14 rules. At least 8–10 should be strong enough for Apriori to discover.

| No. | Existing products | Recommended product | Business logic | Target confidence |
|---:|---|---|---|---:|
| 1 | Pension Account + Pension Card | Pension Scheme | Retirement cross-sell | 80–85% |
| 2 | Savings Account + Debit Card | Credit Card | Existing banking customer upgrade | 35–45% |
| 3 | Salary Account + Credit Card | Personal Loan | Salaried customers are loan prospects | 45–55% |
| 4 | Savings Account + Fixed Deposit | Mutual Fund | Investment cross-sell | 55–65% |
| 5 | Savings Account + Mutual Fund | Demat Account | Investment expansion | 50–60% |
| 6 | Home Loan | Home Insurance | Protect financed property | 60–70% |
| 7 | Vehicle Loan | Life Insurance | Loan and financial protection | 45–55% |
| 8 | Vehicle Loan + Savings Account | Vehicle Insurance | Vehicle-related cross-sell | 60–70% |
| 9 | Education Loan + Savings Account | Credit Card | Young banking customers | 35–45% |
| 10 | Salary Account + Savings Account | Credit Card | Salary customer upgrade | 40–50% |
| 11 | Credit Card + Savings Account | Personal Loan | Credit-product expansion | 35–45% |
| 12 | Fixed Deposit + Savings Account | Recurring Deposit | Savings and investment expansion | 40–50% |
| 13 | High Income + Mutual Fund | Wealth Management Service | Premium customer cross-sell | 55–65% |
| 14 | Mutual Fund + Demat Account | Wealth Management Service | Investment-heavy customer | 60–70% |

### Guaranteed Demo Rule

Keep this as the strongest and most reliable rule:

```text
Pension Account + Pension Card -> Pension Scheme
Confidence: around 82%
Lift: greater than 2.5
```

---

## 9. Dataset Requirements

Generate:

```text
20,000 customers                          (customers.csv - minimum 20k rows)
21 banking products                       (all products needed by the 14 patterns)
~99,600 customer-product ownership rows   (customer_products.csv)
```

20,000 baskets gives stable support / confidence estimates, and Apriori with `min_support=0.03` still runs in seconds.

### Files to Generate

| File | Purpose |
|---|---|
| `customers.csv` | Customer demographic and profile data |
| `products.csv` | Banking product master data |
| `customer_products.csv` | Product ownership transactions |
| `customer_profiles.csv` | Optional hidden profile label for analysis and validation |

### `customers.csv`

| Column | Example |
|---|---|
| customer_id | CUST00001 |
| name | Rahul Sharma |
| email | rahul.sharma4821@gmail.com |
| age | 42 |
| gender | Male |
| income_group | High |
| annual_income | 1850000 |
| occupation | Software Engineer |
| city | Mumbai |
| state | Maharashtra |
| avg_monthly_balance | 184500 |
| credit_score | 742 |
| preferred_channel | Mobile App |
| kyc_status | Verified |
| account_status | Active |
| customer_profile | Salaried |
| created_at | 2023-04-18 |

`customers.csv` has 17 columns. `income_group` is derived from `annual_income` (INR): **Low** < 4,00,000, **Medium** 4,00,000 - 14,99,999, **High** ≥ 15,00,000. `credit_score` (300-900), `annual_income`, `avg_monthly_balance` and `preferred_channel` are included so the eligibility engine and analytics have realistic numeric inputs.

### `products.csv`

| Column | Example |
|---|---|
| product_id | P017 |
| product_name | Pension Scheme |
| category | Pension |
| description | Retirement pension product |
| min_age | 40 |
| max_age | 100 |

### `customer_products.csv`

| Column | Example |
|---|---|
| customer_id | CUST00001 |
| product_id | P001 |
| product_name | Savings Account |
| start_date | 2023-05-02 |
| status | Active |

---

## 10. Banking Products

Use these 21 products:

```text
P001 Savings Account
P002 Salary Account
P003 Current Account
P004 Debit Card
P005 Credit Card
P006 Fixed Deposit
P007 Recurring Deposit
P008 Personal Loan
P009 Home Loan
P010 Education Loan
P011 Vehicle Loan
P012 Life Insurance
P013 Health Insurance
P014 Home Insurance
P015 Vehicle Insurance
P016 Mutual Fund
P017 Pension Account
P018 Pension Card
P019 Pension Scheme
P020 Demat Account
P021 Wealth Management Service
```

All 21 products are used by the 14 cross-sell patterns, so none can be dropped.

---

## 11. Dataset Generation Logic

### How the 20,000-customer generator avoids repetitive data

- **Continuous distributions:** age (truncated normal per profile), annual income (log-normal per profile), average balance (log-normal) and credit score (normal) - not a handful of fixed values.
- **Wide categorical spread:** 30 cities (weighted by size) with states, 8-12 occupations per profile, 5 acquisition channels, 8 e-mail domains and 5 e-mail formats.
- **Unique identifiers:** every e-mail is unique; names come from Faker `en_IN` when installed, otherwise from a built-in pool of Indian first/last names.
- **Blended customers:** 12% of customers also behave like a second profile (e.g. a Salaried customer who invests), giving ~2,600 distinct product baskets.
- **Realistic dates:** account-opening dates skew toward recent years; product start dates fall between the customer's join date and today.
- **Age-eligible ownership:** a product is only assigned if the customer's age is inside the product's min/max age.

### Step 1: Assign Customer Profile

Randomly assign each customer to one of the behavior profiles.

```text
Young / Education: 15%
Salaried: 22%
Investor: 18%
Homeowner: 12%
Vehicle Owner: 10%
Retired / Senior: 13%
High-Value Customer: 10%
General Customer: remaining
```

### Step 2: Assign Base Products

Each profile receives its base products with high probability.

Example:

```text
Retired / Senior customer:
- Savings Account: 95%
- Debit Card: 85%
- Pension Account: 90%
- Pension Card: 80%
```

### Step 3: Apply Cross-Sell Rules

After assigning base products, apply conditional probabilities.

Example:

```python
if "Pension Account" in owned and "Pension Card" in owned:
    add_product("Pension Scheme", probability=0.82)

if "Salary Account" in owned and "Credit Card" in owned:
    add_product("Personal Loan", probability=0.50)

if "Home Loan" in owned:
    add_product("Home Insurance", probability=0.65)

if "Savings Account" in owned and "Fixed Deposit" in owned:
    add_product("Mutual Fund", probability=0.60)
```

### Step 4: Add Noise

Add 0-2 random extra products (about 20% of customers get at least one) and background ownership (e.g. Life / Health Insurance at 12-15%) to make the dataset realistic.

Examples:

```text
Some young customers may have Fixed Deposit.
Some salaried customers may have Health Insurance.
Some homeowners may have Mutual Fund.
```

Noise prevents the data from becoming too artificial.

### Step 5: Validate Rules

After generation, calculate support, confidence, and lift for each intended rule.

The generator prints a validation table after every run. Results for the 20,000-customer dataset (seed 42):

| # | Rule | Target conf. | Achieved conf. | Lift | Customers matched | Status |
|---:|---|---:|---:|---:|---:|:---:|
| 1 | Pension Account + Pension Card -> Pension Scheme | 80-85% | **81.8%** | 11.49 | 1,740 | ✅ |
| 2 | Savings Account + Debit Card -> Credit Card | 35-45% | 35.1% | 1.09 | 17,697 | ✅ (low edge) |
| 3 | Salary Account + Credit Card -> Personal Loan | 45-55% | 56.9% | 3.62 | 1,782 | ≈ (+1.9) |
| 4 | Savings Account + Fixed Deposit -> Mutual Fund | 55-65% | 53.6% | 2.15 | 6,448 | ≈ (-1.4) |
| 5 | Savings Account + Mutual Fund -> Demat Account | 50-60% | 54.6% | 2.67 | 4,938 | ✅ |
| 6 | Home Loan -> Home Insurance | 60-70% | 64.5% | 7.52 | 2,659 | ✅ |
| 7 | Vehicle Loan -> Life Insurance | 45-55% | 49.2% | 2.14 | 2,373 | ✅ |
| 8 | Vehicle Loan + Savings Account -> Vehicle Insurance | 60-70% | 62.5% | 8.56 | 2,336 | ✅ |
| 9 | Education Loan + Savings Account -> Credit Card | 35-45% | 42.9% | 1.33 | 1,542 | ✅ |
| 10 | Salary Account + Savings Account -> Credit Card | 40-50% | 45.6% | 1.41 | 3,897 | ✅ |
| 11 | Credit Card + Savings Account -> Personal Loan | 35-45% | 38.8% | 2.47 | 6,439 | ✅ |
| 12 | Fixed Deposit + Savings Account -> Recurring Deposit | 40-50% | 51.1% | 2.69 | 6,448 | ≈ (+1.1) |
| 13 | High Income + Mutual Fund -> Wealth Management Service | 55-65% | 64.0% | n/a | 2,372 | ✅ |
| 14 | Mutual Fund + Demat Account -> Wealth Management Service | 60-70% | 55.3% | 5.96 | 2,718 | ≈ (-4.7) |

✅ = inside the target range, ≈ = within ~5 points of it. Rules 3, 4, 12 and 14 can be nudged by editing the probabilities in the `PROFILES` / `GLOBAL_RULES` tables of the script.

**Notes on reading this table**

- **Lift of rule 2 is only 1.09.** About 98% of customers own a Savings Account, so almost any rule built on it has low lift even though its confidence looks healthy (see recommendation 4.3).
- **Rule 13 uses income, not a product.** To let Apriori discover it, add an `Income: High` pseudo-item to every high-income customer's basket.
- **Weakest itemset support is ~3.3%** (Education Loan + Savings Account + Credit Card), so `min_support = 0.03` is just enough; use 0.025 for headroom.
- **Guaranteed demo rule:** Pension Account + Pension Card -> Pension Scheme sits at ~82% confidence with lift ≈ 11.5, well above the 2.5 requirement.

---

## 12. Recommendation Engine Logic

For each customer:

1. Load products already owned.
2. Load generated association rules.
3. Find rules where the customer owns all antecedent products.
4. Remove recommended products already owned.
5. Apply eligibility rules.
6. Rank remaining products by confidence, then lift.
7. Return top 1–5 recommendations.

### Example

Customer owns:

```text
Savings Account
Fixed Deposit
Debit Card
```

Matching rule:

```text
Savings Account + Fixed Deposit -> Mutual Fund
Confidence: 61.5%
Lift: 2.20
```

Backend response:

```json
{
  "customer_id": "CUST00452",
  "current_products": [
    "Savings Account",
    "Fixed Deposit",
    "Debit Card"
  ],
  "recommendations": [
    {
      "product": "Mutual Fund",
      "confidence": 0.615,
      "lift": 2.20,
      "eligible": true,
      "reason": "Customers with Savings Account and Fixed Deposit frequently also hold Mutual Fund."
    }
  ]
}
```

---

## 13. Eligibility Rules

| Product | Eligibility condition |
|---|---|
| Pension Scheme | Age 40+ and owns Pension Account |
| Credit Card | Age 21–65 and income not Low |
| Personal Loan | Age 21–60 |
| Home Loan | Age 25–60 and no active Home Loan |
| Education Loan | Age 18–35 |
| Vehicle Loan | Age 21–60 |
| Mutual Fund | KYC Verified |
| Demat Account | KYC Verified |
| Wealth Management Service | High income or owns Mutual Fund + Demat Account |
| Home Insurance | Owns Home Loan or age 25+ |
| Vehicle Insurance | Owns Vehicle Loan |
| Life Insurance | Age 18–70 |
| Health Insurance | Age 18–80 |

### Extended eligibility (uses the new dataset columns)

These thresholds are suggested starting points - tune them to your own bank policy.

| Product | Additional condition |
|---|---|
| Credit Card | `credit_score` ≥ 650 (plus the age / income rule above) |
| Personal Loan | `credit_score` ≥ 650 |
| Vehicle Loan | `credit_score` ≥ 650 |
| Home Loan | `credit_score` ≥ 700 |
| Mutual Fund, Demat Account, Wealth Management Service | `kyc_status = Verified` |
| Wealth Management Service | `annual_income` ≥ 15,00,000 (High income) or owns Mutual Fund + Demat Account |

**Global rules**

- Customers with `account_status = Inactive` are excluded from recommendations sent as campaigns (they may still be viewed on the profile page, marked ineligible).
- Customers with `kyc_status = Pending` are never offered investment products.
- Every recommendation response returns `eligible: true/false` and a short `ineligible_reason` so the UI can explain why.

The generator assigns products by age eligibility only, so a few customers already hold products that the stricter credit-score rules would now block. This only affects *new* recommendations, not existing holdings.

---

## 14. Email Campaign Feature

The email feature must be generic so it works for every product recommendation.

### Email Flow

```text
Customer owns products
        |
        v
Backend finds recommendation
        |
        v
Employee clicks “Send Offer Email”
        |
        v
Backend selects product-specific email template
        |
        v
Personalized email is sent through SMTP
        |
        v
Campaign log is saved
```

### Example: Pension Scheme Email

```text
Subject: Exclusive Pension Scheme Offer for You

Dear {{customer_name}},

Based on your existing Pension Account and Pension Card,
you may be eligible for our Pension Scheme.

Benefits:
- Guaranteed monthly pension
- Flexible contribution options
- Tax benefits
- Dedicated senior-citizen support

Visit your nearest branch or reply to this email to know more.

Regards,
XYZ Bank
```

### Example: Mutual Fund Email

```text
Subject: Start Your Investment Journey with Mutual Funds

Dear {{customer_name}},

You already maintain a Savings Account and Fixed Deposit with XYZ Bank.
Based on your savings pattern, you may be interested in our Mutual Fund options.

Benefits:
- Professional fund management
- Flexible investment amounts
- Potential long-term wealth creation

Regards,
XYZ Bank
```

### Email Testing

Use your own Gmail address as the recipient during testing.

Create a `.env` file:

```env
MAIL_SERVER=smtp.gmail.com
MAIL_PORT=587
MAIL_USE_TLS=True
MAIL_USERNAME=your_test_email@gmail.com
MAIL_PASSWORD=your_gmail_app_password
MAIL_DEFAULT_SENDER=your_test_email@gmail.com
TEST_MODE=True
TEST_RECIPIENT=your_test_email@gmail.com
```

Do not commit `.env` to GitHub.

### Safe Sending: TEST_MODE, Dry-Run and Logging

The generated e-mail addresses (e.g. `name1234@gmail.com`) may belong to real people, so **no demo mail should ever be sent to them.**

```python
import os

def resolve_recipient(customer_email: str) -> str:
    # In test mode every mail goes to your own inbox
    if os.getenv("TEST_MODE", "True") == "True":
        return os.getenv("TEST_RECIPIENT")
    return customer_email

def build_subject(subject: str, customer_email: str) -> str:
    if os.getenv("TEST_MODE", "True") == "True":
        return f"[TEST -> {customer_email}] {subject}"
    return subject
```

- **TEST_MODE (default ON):** all mail is redirected to `TEST_RECIPIENT`; the intended customer appears in the subject line and body.
- **Dry-run:** `POST /api/campaigns/<id>/send?dry_run=true` renders every personalised e-mail and writes the log, but sends nothing.
- **Logging:** every attempt is stored in `campaign_logs` with the intended recipient, the actual recipient, mode (test / live / dry-run), status and timestamp.
- **Optional extra safety:** change `EMAIL_DOMAINS` in the generator to `example.com` before building the dataset for a live demo.

---

## 15. Database Design

### `customers`

```text
customer_id
name
email
age
gender
income_group
annual_income
occupation
city
state
avg_monthly_balance
credit_score
preferred_channel
kyc_status
account_status
customer_profile
created_at
```

### `products`

```text
product_id
product_name
category
description
min_age
max_age
```

### `customer_products`

```text
id
customer_id
product_id
product_name
start_date
status
```

### `association_rules`

```text
rule_id
antecedents
consequents
support
confidence
lift
created_at
```

### `campaigns`

```text
campaign_id
campaign_name
target_product
email_subject
email_body
created_at
status
```

### `campaign_logs`

```text
log_id
campaign_id
customer_id
email_status
email_mode          (test / live / dry_run)
intended_recipient
actual_recipient
sent_at
customer_response
```

### `evaluation_results`

```text
run_id
evaluated_at
top_k
test_customers
hit_rate_rules
precision_rules
hit_rate_popularity_baseline
min_support
min_confidence
min_lift
```

### `rule_validation`

```text
validation_id
run_id
planted_rule
target_confidence
generated_confidence
discovered            (true / false)
discovered_confidence
discovered_lift
```

---

## 16. Backend API Endpoints

Base URL:

```text
http://localhost:5000/api
```

### Authentication

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/auth/login` | Admin login |
| POST | `/api/auth/logout` | Admin logout |
| GET | `/api/auth/me` | Current admin details |

### Customers

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/customers` | List and search customers |
| GET | `/api/customers/<customer_id>` | Customer profile |
| POST | `/api/customers` | Create customer |
| PUT | `/api/customers/<customer_id>` | Update customer |
| DELETE | `/api/customers/<customer_id>` | Delete customer |
| GET | `/api/customers/<customer_id>/products` | Products owned by customer |
| POST | `/api/customers/<customer_id>/products` | Add product ownership |
| DELETE | `/api/customers/<customer_id>/products/<product_id>` | Remove product ownership |

### Products

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/products` | List products |
| GET | `/api/products/<product_id>` | Product details |
| POST | `/api/products` | Create product |
| PUT | `/api/products/<product_id>` | Update product |
| DELETE | `/api/products/<product_id>` | Delete product |

### Market Basket Analysis

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/analysis/run` | Run Apriori and generate rules |
| GET | `/api/analysis/rules` | Get all association rules |
| GET | `/api/analysis/frequent-itemsets` | Get frequent itemsets |
| POST | `/api/analysis/parameters` | Set support, confidence, and lift |
| GET | `/api/analysis/validation` | Planted rules vs rules discovered by Apriori (recovery rate) |
| POST | `/api/analysis/evaluation` | Run the hide-one hold-out test (hit rate / precision@k vs baseline) |
| GET | `/api/analysis/evaluation` | Latest evaluation results |

Request example:

```json
{
  "min_support": 0.03,
  "min_confidence": 0.45,
  "min_lift": 1.05
}
```

> **Why 1.05?** Savings Account is owned by ~98% of customers, so rules built on it (e.g. Savings Account + Debit Card -> Credit Card, lift ≈ 1.09) can never reach a high lift. A threshold of 1.2 hides them. Use `min_confidence: 0.35` and `min_lift: 1.05` to recover all 14 planted rules, or `0.45` / `1.2` for a cleaner "top rules only" demo.

### Recommendations

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/customers/<customer_id>/recommendations` | Get all valid recommendations |
| GET | `/api/customers/<customer_id>/next-best-product` | Get top recommendation |
| POST | `/api/recommendations/bulk` | Get recommendations for selected customers |

### Campaigns and Emails

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/campaigns` | List campaigns |
| POST | `/api/campaigns` | Create campaign |
| GET | `/api/campaigns/<campaign_id>` | Campaign details |
| POST | `/api/campaigns/<campaign_id>/send` | Send campaign emails (honours `TEST_MODE`; supports `?dry_run=true`) |
| POST | `/api/campaigns/<campaign_id>/send-test` | Send one test email |
| GET | `/api/campaigns/<campaign_id>/logs` | Email delivery logs |
| PUT | `/api/campaigns/logs/<log_id>/feedback` | Save customer response |

### Analytics

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/analytics/product-popularity` | Most common products |
| GET | `/api/analytics/product-pairs` | Most frequent product pairs |
| GET | `/api/analytics/top-rules` | Best rules by confidence/lift |
| GET | `/api/analytics/customer-profiles` | Distribution by customer profile |
| GET | `/api/analytics/campaign-summary` | Campaign performance |
| GET | `/api/analytics/recommendation-summary` | Most recommended products |

---

## 17. Frontend Pages

### 1. Login Page

**Route:** `/login`

- Admin email and password
- Login button
- Error handling

### 2. Dashboard

**Route:** `/dashboard`

- Total customers
- Total products
- Total product holdings
- Total association rules
- Total campaigns
- Emails sent
- Top recommended products
- Quick navigation

### 3. Customers Page

**Route:** `/customers`

- Search by ID, name, email
- Filter by age, income, city, profile
- Customer table
- Add/edit/delete customer
- Open customer profile

### 4. Customer Profile Page

**Route:** `/customers/<customer_id>`

Show:

- Customer details
- Customer behavior profile
- Products currently owned
- Recommended products
- Confidence and lift
- Eligibility status
- Recommendation reason
- Campaign history
- Buttons:
  - Get Recommendations
  - Send Offer Email
  - Add Product
  - Edit Customer

Example UI:

```text
CUSTOMER: CUST00452

Current Products
----------------
✓ Savings Account
✓ Fixed Deposit
✓ Debit Card

Recommended Next Product
------------------------
⭐ Mutual Fund

Confidence: 61.5%
Lift: 2.20

Why?
Customers with Savings Account + Fixed Deposit
frequently also hold Mutual Fund.

[ Send Offer Email ]
```

### 5. Products Page

**Route:** `/products`

- Product list
- Add/edit/delete product
- Filter by category

### 6. Customer Products Page

**Route:** `/customers/<customer_id>/products`

- Products owned
- Add product
- Remove product
- Product status

### 7. Association Rules Page

**Route:** `/analysis/rules`

- Run Apriori
- Set minimum support
- Set minimum confidence
- Set minimum lift
- View rules table
- Filter by product
- Show leverage and conviction columns
- Validation panel: planted rules vs discovered rules
- Export rules to CSV

### 8. Frequent Itemsets Page

**Route:** `/analysis/frequent-itemsets`

- Frequent product combinations
- Sort by support
- Filter by itemset size
- Charts

### 9. Campaigns Page

**Route:** `/campaigns`

- Create campaign
- Select target product
- Select customers
- Choose email template
- Preview email
- Send test email
- Send bulk campaign
- Dry-run button
- TEST_MODE banner (shows where mail is really going)
- View campaign status

### 10. Campaign Details Page

**Route:** `/campaigns/<campaign_id>`

- Campaign information
- Selected customers
- Email template
- Delivery status
- Sent time
- Customer response
- Retry failed emails

### 11. Analytics Page

**Route:** `/analytics`

- Product popularity chart
- Frequent product-pair chart
- Support vs confidence scatter plot
- Top rules by lift
- Customer-profile distribution
- Most recommended products
- Product-pair co-ownership heatmap
- Rule strength (lift) by customer profile
- Recommender evaluation: hit rate / precision@k vs popularity baseline

---

## 18. Frontend Route Summary

| Page | Route |
|---|---|
| Login | `/login` |
| Dashboard | `/dashboard` |
| Customers | `/customers` |
| Customer Profile | `/customers/<id>` |
| Products | `/products` |
| Customer Products | `/customers/<id>/products` |
| Association Rules | `/analysis/rules` |
| Frequent Itemsets | `/analysis/frequent-itemsets` |
| Campaigns | `/campaigns` |
| Campaign Details | `/campaigns/<id>` |
| Analytics | `/analytics` |

---

## 19. Project Folder Structure

```text
banking-cross-sell-system/
│
├── backend/
│   ├── app.py
│   ├── config.py
│   ├── models.py
│   ├── requirements.txt
│   ├── .env
│   │
│   ├── routes/
│   │   ├── auth_routes.py
│   │   ├── customer_routes.py
│   │   ├── product_routes.py
│   │   ├── analysis_routes.py
│   │   ├── recommendation_routes.py
│   │   ├── campaign_routes.py
│   │   └── analytics_routes.py
│   │
│   ├── services/
│   │   ├── mba_service.py
│   │   ├── recommendation_service.py
│   │   ├── eligibility_service.py
│   │   ├── email_service.py
│   │   ├── analytics_service.py
│   │   └── evaluation_service.py
│   │
│   ├── utils/
│   │   ├── validators.py
│   │   └── email_templates.py
│   │
│   ├── data/
│   │   ├── customers.csv
│   │   ├── products.csv
│   │   ├── customer_products.csv
│   │   └── customer_profiles.csv
│   │
│   └── database/
│       └── banking_mba.db
│
├── frontend/
│   ├── index.html
│   ├── login.html
│   ├── dashboard.html
│   ├── customers.html
│   ├── customer_profile.html
│   ├── products.html
│   ├── rules.html
│   ├── frequent_itemsets.html
│   ├── campaigns.html
│   ├── campaign_details.html
│   ├── analytics.html
│   │
│   ├── css/
│   │   └── style.css
│   │
│   └── js/
│       ├── api.js
│       ├── auth.js
│       ├── customers.js
│       ├── products.js
│       ├── rules.js
│       ├── campaigns.js
│       └── analytics.js
│
├── scripts/
│   └── generate_banking_dataset.py
│
├── notebooks/
│   ├── generate_dataset.ipynb
│   ├── market_basket_analysis.ipynb
│   ├── rule_validation.ipynb
│   └── recommender_evaluation.ipynb
│
├── docs/
│   ├── project_report.md
│   ├── api_documentation.md
│   └── presentation.pptx
│
└── README.md
```

---

## 20. Development Plan

### Recommended build order

1. Dataset generator ✅
2. Apriori + recommendation engine, tested **from the command line / notebook first**
3. Rule validation and recommender evaluation
4. Flask APIs and SQLite
5. Frontend (one clean dashboard beats a half-finished full UI)
6. Email campaigns last (in TEST_MODE)

Keep login simple: a single admin account is enough.

### Phase 1: Dataset Generator

**Duration: 2 days** - ✅ completed (20,000 customers, ~99,600 ownership rows)

- Create product master list
- Create customer profiles
- Generate customers using Faker
- Assign base products by profile
- Apply 14 cross-sell rules
- Add realistic noise
- Save CSV files
- Validate support, confidence, and lift

### Phase 2: Backend Foundation

**Duration: 2–3 days**

- Create Flask app
- Configure SQLite
- Create SQLAlchemy models
- Import CSV data into database
- Build customer and product APIs
- Build customer-product ownership APIs

### Phase 3: Market Basket Analysis

**Duration: 2 days**

- Load ownership data
- Create customer baskets
- Apply TransactionEncoder
- Run Apriori
- Generate association rules
- Store rules in database
- Build rule APIs
- Run rule validation: planted rules vs discovered rules (include `Income: High` pseudo-item for rule 13)
- Add leverage and conviction to the stored rules

### Phase 4: Recommendation Engine

**Duration: 2 days**

- Match customer-owned products with rule antecedents
- Remove already-owned products
- Apply eligibility checks
- Rank by confidence and lift
- Build recommendation APIs
- Add extended eligibility (credit score, KYC, account status)
- Run hide-one evaluation (hit rate / precision@3) against a popularity baseline

### Phase 5: Email Campaign Module

**Duration: 2 days**

- Configure Gmail SMTP
- Create product-specific templates
- Build campaign APIs
- Implement single and bulk email sending
- Save campaign logs
- Add TEST_MODE, dry-run and recipient logging
- Test with your own email address

### Phase 6: Frontend

**Duration: 3–4 days**

- Build login and dashboard
- Build customer pages
- Build product pages
- Build rules and frequent-itemsets pages
- Build recommendation UI
- Build campaign pages
- Add analytics charts

### Phase 7: Testing and Documentation

**Duration: 2 days**

- Test APIs in Postman
- Test frontend-backend integration
- Test all major recommendation patterns
- Test email sending
- Prepare report, PPT, and demo script

### Phase 8 (Optional): Rule-Based vs Machine-Learning Comparison

**Duration: 2-3 days**

- Train logistic regression and a decision tree to predict ownership of a target product from age, income, credit score and owned products
- Compare AUC / hit rate@3 with the Apriori recommender
- Add the comparison to the report

---

## 21. Testing Plan

### Main Test Cases

| Test | Expected result |
|---|---|
| Customer owns Pension Account + Pension Card | Pension Scheme is recommended |
| Customer owns Salary Account + Credit Card | Personal Loan is recommended |
| Customer owns Home Loan | Home Insurance is recommended |
| Customer owns Savings Account + Fixed Deposit | Mutual Fund is recommended |
| Customer owns Mutual Fund + Demat Account | Wealth Management Service is recommended |
| Customer already owns recommended product | Product is not recommended |
| Customer is below product age limit | Product is marked ineligible |
| Customer has no matching rule | System returns no recommendation or fallback offer |
| Test email is sent | Email arrives in test inbox |
| Campaign is sent | Campaign log status becomes Sent |
| TEST_MODE is on | Mail arrives only at `TEST_RECIPIENT`; subject shows the intended customer |
| Dry-run campaign | Emails rendered and logged, none sent |
| Rule validation is run | All planted rules are listed with discovered / not discovered status |
| Hide-one evaluation is run | Hit rate @3 of the rule engine is higher than the popularity baseline |
| Customer has credit score below 650 | Credit Card / Personal Loan marked ineligible |

### Edge Cases

- Invalid customer ID
- Customer with zero products
- Customer with inactive account
- Customer with pending KYC requesting Mutual Fund
- Invalid email address
- Wrong SMTP password
- Apriori run before data exists
- Campaign with no selected customers
- Customer with pending KYC offered Wealth Management Service
- `TEST_MODE` missing from `.env` (must default to safe mode)

---

## 22. Demo Script

1. Log in to the admin dashboard.
2. Show total customers, products, rules, campaigns, and emails sent.
3. Open the Association Rules page.
4. Run Apriori with minimum support 0.03, confidence 0.35, and lift 1.05 (use 0.45 / 1.2 for a cleaner top-rules view).
5. Show discovered rules:
   - Pension Account + Pension Card → Pension Scheme
   - Salary Account + Credit Card → Personal Loan
   - Home Loan → Home Insurance
   - Savings Account + Fixed Deposit → Mutual Fund
   - Mutual Fund + Demat Account → Wealth Management Service
6. Open a Retired/Senior customer profile.
7. Show that Pension Scheme is recommended.
8. Click **Send Offer Email** (TEST_MODE redirects it to your own inbox).
9. Open the test Gmail inbox and show the received email with the intended customer in the subject.
10. Open an Investor customer profile.
11. Show that Mutual Fund or Demat Account is recommended.
12. Open Analytics and show product popularity, frequent pairs, the co-ownership heatmap, and top rules.
13. Show the rule-validation table (planted vs discovered rules).
14. Show the recommender evaluation: hit rate / precision@3 vs the popularity baseline.
15. Explain how the bank can use these insights for targeted cross-selling.

---

## 23. Report Structure

1. Title Page
2. Abstract
3. Introduction
4. Problem Statement
5. Objectives
6. Literature Review
7. System Architecture
8. Technology Stack
9. Synthetic Dataset Generation
10. Customer Behavior Profiles
11. Cross-Sell Pattern Design
12. Data Preprocessing
13. Market Basket Analysis
14. Apriori Algorithm
15. Association Rule Metrics
16. Recommendation Engine
17. Eligibility Engine
18. Email Campaign Module
19. Database Design
20. API Documentation
21. Frontend Screens
22. Testing and Results
23. Rule Validation and Recommender Evaluation
24. Business Insights
25. Ethical and Data-Privacy Considerations (Synthetic Data, Test Mode)
26. Limitations
27. Future Scope
28. Conclusion
29. References

---

## 24. Future Scope

- Use real anonymized banking data
- Add customer login and self-service recommendations
- Add SMS and WhatsApp campaigns
- Add collaborative filtering
- Compare the rule-based recommender with supervised models (logistic regression, decision tree, gradient boosting)
- Add customer churn prediction
- Use credit score and transaction history for eligibility
- Deploy backend on Render, Railway, or PythonAnywhere
- Deploy frontend on Vercel or Netlify
- Add role-based access for managers and marketing staff
- Track email opens and clicks using an email provider

---

## 25. Final Deliverables

- Synthetic dataset generator script (`generate_banking_dataset.py`, 20,000 customers)
- `customers.csv`
- `products.csv`
- `customer_products.csv`
- Optional `customer_profiles.csv`
- Flask backend with REST APIs
- SQLite database
- Apriori-based MBA module
- Multi-product recommendation engine
- Eligibility-check module (age, income, credit score, KYC)
- Rule-validation and recommender-evaluation notebooks / reports
- Automated email campaign module
- Frontend admin dashboard
- Analytics dashboard with charts
- API documentation
- Project report
- Presentation
- Live demo or demo video
#   M L - M i n i _ P r o j e c t  
 