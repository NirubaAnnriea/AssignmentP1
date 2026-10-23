
# loan-calc Architecture

## Overview
This repository provides the emi calculation utilities. The core logic lives in the calculator.py file, which exposes small, composable functions for:
- computing EMI and totals,
- generating amortization schedules,
- estimating remaining tenure after prepayment,
- comparing offers,
- formatting INR values,
- producing a structured summary and exporting schedules to CSV.

---

### The constant
- **`DEFAULT_CURRENCY = "INR"`**
  - It is used by `summary()` to label the currency in the returned payload.

### Function reference (behavior + purpose)

#### `_monthly_rate(annual_rate)`
- It is to convert an **annual interest rate in percent** into a **monthly decimal rate**.
- The implementation is `round(annual_rate / 1200, 4)`
  - `/1200` = `/100` (percent→decimal) and `/12` (annual→monthly).

---

#### `emi(principal, annual_rate, tenure_months)`
- It calculates the **fixed monthly EMI** for a loan.
- The validations will raise `ValueError` if it is not positive:
    - `tenure_months <= 0`
    - `principal <= 0`
- Uses:
  - `r = _monthly_rate(annual_rate)`
- Cases:
  - If `r == 0`: `principal / tenure_months`
  - Else standard EMI formula:
    - \[
      EMI = P \cdot \frac{r(1+r)^n}{(1+r)^n - 1}
      \]
- It returns an emi amount that is rounded to **2 decimals**.

---

#### `total_payment(principal, annual_rate, tenure_months)`
- This function is to calculate the total cash paid across the full tenure.
- Logic: `emi(...) * tenure_months`
- It returns the totaly payment rounded to **2 decimals**.

---

#### `total_interest(principal, annual_rate, tenure_months)`
- This function is to calculate the total interest paid over the tenure.
- Logic: `total_payment(...) - principal`
- It return the total interest rounded to **2 decimals**.

---

#### `amortisation_schedule(principal, annual_rate, tenure_months)`
- The function produces a list of per-month amortization rows, each containing:
  - `month`
  - `emi`
  - `interest` (interest portion for that month)
  - `principal` (principal portion for that month)
  - `balance` (remaining balance after that month)
- Steps:
  - `r = _monthly_rate(annual_rate)`
  - `instalment = emi(principal, ...)`
  - Start `balance = principal`
  - For each month, calculate:
    - `interest = round(balance * r, 2)`
    - `principal_part = round(instalment - interest, 2)`
    - reduce `balance` accordingly
- Returns: `rows` (a `list[dict]`).

> To note: The current loop is `range(1, tenure_months)`, so it iterates only `tenure_months - 1` times. There is a special-case block intended for the final month (`if month == tenure_months:`) but it is unreachable with the current loop bounds.

---

#### `outstanding_after(principal, annual_rate, tenure_months, months_paid)`
- This function returns the outstanding balance after paying for `months_paid`.
- Behavior:
  - If `months_paid <= 0`: returns `principal`
  - Builds amortization schedule via `amortisation_schedule(...)`
  - If `months_paid > len(sched)`: returns `0.0`
  - Else returns `sched[months_paid - 1]["balance"]`

---

#### `remaining_tenure(balance, annual_rate, instalment)`
- It calculates how many additional months are needed to repay `balance` given a fixed monthly installment.
- Logic:
  - `r = annual_rate / 100`
  - If `balance <= 0`: `0`
  - If `r == 0`: `ceil(balance / instalment)`
  - If `instalment <= balance * r`: raise `ValueError("instalment too small to repay the loan")`
  - Else uses a logarithmic closed-form and rounds up with `ceil`.
- Returns: `ceil(n)` (months as an integer)

---

#### `prepayment_savings(principal, annual_rate, tenure_months, prepay_month, prepay_amount)`
- Models a **lump-sum prepayment** that occurs at `prepay_month`, assuming EMI stays the same and tenure shortens.
- Flow:
  1. Compute original `instalment = emi(...)`
  2. Build full amortization schedule
  3. Determine balance at `prepay_month`:
     - `balance = sched[prepay_month - 1]["balance"]`
  4. Apply prepayment:
     - `new_balance = balance - prepay_amount`
  5. If `new_balance <= 0`: set remaining months to 0
     - else compute remaining tenure via `remaining_tenure(...)`
  6. Compute:
     - months saved
     - interest paid before prepayment
     - new projected total interest
     - interest saved = original_interest - new_interest
- Returns: dict with keys:
  - `emi`
  - `months_saved`
  - `new_tenure`
  - `interest_saved`
  - `interest_paid_before_prepayment`

---

#### `compare_offers(principal, offers)`
- It compares multiple loan offers and ranks them by lowest total cost.
- Expected offer fields:
  - `bank`
  - `rate` (annual rate)
  - `tenure` (months)
  - optional `processing_fee_pct`
- Per offer:
  - compute `emi` and `total_interest`
  - compute `fee = processing_fee_pct * principal / 100`
  - compute `total_cost = total_interest + fee`
- Sorts by `total_cost` ascending.
- Returns: list of dicts:
  - `bank`, `emi`, `total_interest`, `processing_fee`, `total_cost`

---

#### `eligible_amount(monthly_income, existing_emis, annual_rate, tenure_months, foir=0.5)`
- It estimates the maximum loan principal based on affordability using FOIR.
- Logic:
  - `available = monthly_income * foir - existing_emis`
  - If `available <= 0`: return `0.0`
  - Convert annual rate to monthly rate using `_monthly_rate(annual_rate)`
  - If `r == 0`: `available * tenure_months`
  - Else solve inverse EMI formula for principal given EMI=available:
    - uses `f = (1 + r) ** tenure_months`
    - `P = available * (f - 1) / (r * f)`
- Returns: rounded to **2 decimals**.

---

#### `format_inr(amount)`
- It formats a number into Indian currency formatting using ₹ and comma grouping.
- It handles negative values by prefixing `-`.
- It ensures two decimal places.
- Returns: formatted string like `₹12,34,567.89`.

---

#### `schedule_to_csv(rows, path)`
- Exports amortization schedule rows to a CSV file.
- Output columns:
  - `month,emi,interest,principal,balance`
- Returns: nothing (writes file at `path`).

---

#### `summary(principal, annual_rate, tenure_months, start=None)`
- It creates a summary payload for a loan offer.
- Calculates:
  - EMI (`emi(...)`)
  - total interest (`total_interest(...)`)
  - total payment (`total_payment(...)`)
  - `last_emi` month boundary using `start` date (default: today)
- Formats:
  - principal / totals via `format_inr`
  - rate string as `"{annual_rate}% p.a."`
- Returns: dict with keys:
  - `principal`, `rate`, `tenure`, `emi`, `total_interest`, `total_payment`, `last_emi`, `currency`

---

## How data flows (high level)
- **Core math:** `_monthly_rate()` → used by `emi()` and `eligible_amount()`
- **Aggregations:** `total_payment()` and `total_interest()` build on `emi()`
- **Schedules:** `amortisation_schedule()` drives `outstanding_after()` and `prepayment_savings()`
- **Comparisons:** `compare_offers()` repeatedly calls `emi()`/`total_interest()`
- **Presentation/IO:** `summary()` formats results; `format_inr()` and `schedule_to_csv()` handle formatting/output.
