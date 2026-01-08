# Student Loan Repayment Optimizer
**Author:** Ramadan Raji | MS Business Analytics, UW-Madison

A Mixed Integer Programming model that determines the optimal payment strategy for student loans, minimizing total interest paid over time.

**Result:** $1,279 saved (20.7% less interest) compared to a naive equal-payment strategy.

![Python](https://img.shields.io/badge/Python-3.x-3776ab?style=flat-square&logo=python&logoColor=white)
![Pyomo](https://img.shields.io/badge/Pyomo-Optimization-00a86b?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1HwCoyNCm23zXjMjIbdqI6ey0ObKr3xTc?usp=sharing)

---

## Table of Contents

- [Background](#background)
- [The Problem](#the-problem)
- [Approach](#approach)
- [Results](#results)
- [Model Formulation](#model-formulation)
- [Usage](#usage)
- [Limitations and Future Work](#limitations-and-future-work)
- [About](#about)

---

## Background

Student loan repayment is a challenge millions of graduates face. With multiple loans at different interest rates, some subsidized and some not, deciding how to allocate limited monthly payments is not straightforward.

Common strategies include:
- **Avalanche method:** Pay highest interest rate first
- **Snowball method:** Pay smallest balance first
- **Equal split:** Divide payments equally across all loans

But none of these account for the unique behavior of federal student loans, where subsidized loans do not accrue interest while the borrower is in school. This creates an opportunity for optimization.

---

## The Problem

I have six federal student loans:

| Loan | Balance | Interest Rate | Type |
|------|---------|---------------|------|
| 1-01 | $3,522 | 4.29% | Subsidized |
| 1-03 | $3,518 | 3.76% | Subsidized |
| 1-05 | $2,054 | 4.45% | Subsidized |
| 1-06 | $1,204 | 5.05% | Subsidized |
| 1-07 | $12,528 | 7.94% | Unsubsidized |
| 1-08 | $10,295 | 7.94% | Unsubsidized |

**Total debt:** $33,121

**Key constraints:**
- Budget of $400/month while in school (through May 2026)
- Budget of $800/month after graduation
- Subsidized loans do not accrue interest while enrolled
- Unsubsidized loans accrue interest immediately
- Federal minimum payment requirements apply

**Question:** How should I allocate my payments each quarter to minimize total interest paid?

---

## Approach

I formulated this as a Mixed Integer Programming (MIP) problem:

**Decision Variables:**
- `x[i,t]`: Amount paid toward loan i in quarter t (continuous)
- `y[i,t]`: Whether loan i is fully paid off by quarter t (binary)

**Objective:**
Minimize the sum of all interest accrued across all loans and all time periods.

**Why MIP?**
The binary payoff indicator variables allow the model to track when each loan is eliminated and enforce constraints like "minimum payments only apply while the loan is active." This conditional logic requires integer variables.

**Why quarterly instead of monthly?**
A monthly model (60 periods) took several minutes to solve. Quarterly periods (20 periods) solve in under one second with minimal loss of precision.

---

## Results

| Metric | Optimal Strategy | Equal Split Strategy |
|--------|------------------|----------------------|
| Total Interest Paid | $4,913 | $6,192 |
| Total Amount Paid | $38,034 | $39,313 |
| Time to Payoff | Q17 (4.25 years) | Q20 (5 years) |

### Savings: $1,279 in interest (20.7% reduction)

---

### Optimal Payment Strategy

**Phase 1: While in School (Jan 2026 to Jun 2026)**
- Direct all $400/month toward Loan 1-07 (7.94% unsubsidized)
- Pay nothing on subsidized loans (they are not accruing interest)

**Phase 2: After Graduation (Jul 2026 to Jun 2029)**
- Direct most of the $800/month budget toward both unsubsidized loans
- Pay only the required minimums on subsidized loans

**Phase 3: Final Stretch (Jul 2029 to Mar 2030)**
- Once unsubsidized loans are cleared, pay off remaining subsidized loans
- These have lower rates (3.76% to 5.05%) and are less urgent

---

### Why This Works

The naive strategy wastes money by paying down subsidized loans while they are frozen at 0% interest. Meanwhile, the unsubsidized loans at 7.94% continue to grow. The optimal strategy exploits the interest-free period on subsidized loans by aggressively attacking the expensive debt first.

---

## Model Formulation

<details>
<summary><b>Click to expand technical details</b></summary>

### Sets
- `I = {1, 2, 3, 4, 5, 6}`: Set of loans
- `T = {1, 2, ..., 20}`: Set of quarters (5-year horizon)

### Parameters
| Parameter | Description |
|-----------|-------------|
| `B[i]` | Starting balance of loan i |
| `r[i]` | Quarterly interest rate of loan i |
| `sub[i]` | 1 if loan i is subsidized, 0 otherwise |
| `T1` | Last quarter of Phase 1 (in school) |
| `M1` | Quarterly budget during Phase 1 |
| `M2` | Quarterly budget during Phase 2 |

### Variables
| Variable | Type | Description |
|----------|------|-------------|
| `x[i,t]` | Continuous | Payment to loan i in quarter t |
| `y[i,t]` | Binary | 1 if loan i paid off by end of quarter t |
| `L[i,t]` | Continuous | Balance of loan i at end of quarter t |
| `Interest[i,t]` | Continuous | Interest accrued on loan i in quarter t |

### Objective Function
```
Minimize: Σ Σ Interest[i,t]  for all i ∈ I, t ∈ T
```

### Constraints

| Constraint | Description | Count |
|------------|-------------|-------|
| Interest Calculation | Subsidized loans have 0% interest during Phase 1 | 120 |
| Balance Evolution | Balance = Previous Balance + Interest - Payment | 120 |
| Budget Limit | Total quarterly payments cannot exceed budget | 20 |
| Maximum Payment | Cannot pay more than current balance + interest | 120 |
| Payoff Tracking | If y[i,t]=1, then L[i,t]=0 (Big-M formulation) | 120 |
| Payoff Permanence | Once paid off, stays paid off: y[i,t] >= y[i,t-1] | 114 |
| Terminal Condition | All loans paid by end of horizon: L[i,20]=0 | 6 |
| Minimum Payment | Required minimum while loan is active (linearized) | 108 |

**Total: 728 constraints**

### Solver Performance
- Solver: CBC (via IDAES)
- Solution time: < 1 second
- Optimality gap: < 1%

</details>

---

## Usage

### Run in Google Colab

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1HwCoyNCm23zXjMjIbdqI6ey0ObKr3xTc?usp=sharing)

### Customizing for Your Loans

Modify the parameters in Cell 3 of the notebook:

```python
starting_balance = {
    1: 3522.00,   # Your loan 1 balance
    2: 3518.00,   # Your loan 2 balance
    # ... add more loans as needed
}

quarterly_rates = {
    1: 0.0429 / 4,  # Annual rate divided by 4
    2: 0.0376 / 4,
    # ...
}

subsidized = {
    1: 1,  # 1 = subsidized, 0 = unsubsidized
    2: 1,
    # ...
}

model.M1 = Param(initialize=1200)  # Quarterly budget (Phase 1)
model.M2 = Param(initialize=2400)  # Quarterly budget (Phase 2)
```

---

## Limitations and Future Work

### Current Limitations

1. **Fixed income assumption:** The model assumes constant budgets within each phase. In reality, income may grow over time.

2. **No interest capitalization:** Unpaid interest adding to principal after the grace period is not fully modeled.

3. **No tax benefits:** The student loan interest deduction (up to $2,500/year) is not incorporated.

4. **Quarterly granularity:** Monthly payment timing within a quarter is not optimized.

### Potential Extensions

- **Income growth scenarios:** Model increasing budgets over time
- **Refinancing decisions:** Add binary variable for refinancing (lower rate vs. losing federal protections)
- **PSLF modeling:** For public sector workers, optimize for 120 qualifying payments plus forgiveness
- **Web application:** Build an interface where users can input their loans and receive an optimized plan

---

## About

**Author:** Ramadan Raji  
**Program:** MS Business Analytics, University of Wisconsin-Madison  
**Course:** GENBUS 730 (Prescriptive Analytics)

This project was developed as a final assignment for my Prescriptive Analytics course. We had the freedom to choose any optimization problem, and I chose to work on something I face personally: paying off my student loans.

My goal was to apply what I learned in class to a real problem with real stakes. In class, we shifted from a "data first" approach to a "model first, then data" approach. This project gave me the opportunity to experience that shift firsthand. I defined the problem mathematically, built the model, and then plugged in my actual loan data from Federal Student Aid.

The result is a tool that not only helps me make better financial decisions but could help other students facing the same challenge.

---

## License

MIT License. See [LICENSE](LICENSE) for details.
