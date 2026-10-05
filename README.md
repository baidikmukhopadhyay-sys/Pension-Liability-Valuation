# Pension-Liability-Valuation


"""
Pension Liability Valuation & Sensitivity Analysis
Self-directed Actuarial Project
Includes: demographic assumptions, mortality, sensitivity testing, de-risking
"""

import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

np.random.seed(42)

# ============================================================
# 1. ASSUMPTIONS
# ============================================================
N_EMPLOYEES       = 500
RETIREMENT_AGE    = 60
BENEFIT_FACTOR    = 0.015        # 1.5% per year of service
SALARY_GROWTH     = 0.06         # 6%
DISCOUNT_RATE     = 0.07         # 7% base case
VESTING_YEARS     = 5
TURNOVER_RATE     = 0.05
MORTALITY_RATE    = 0.005        # 0.5% annual mortality (simplified)
PLAN_ASSETS       = 50_00_00_000 # ₹50 crore

# ============================================================
# 2. GENERATE EMPLOYEE DATA (Synthetic)
# ============================================================
ages           = np.random.randint(25, 61, N_EMPLOYEES)
years_service  = np.minimum(ages - 22, np.random.randint(1, 25, N_EMPLOYEES))
current_salary = np.random.normal(600000, 150000, N_EMPLOYEES).clip(250000, 1500000)

df = pd.DataFrame({
    "EmployeeID":     range(1, N_EMPLOYEES + 1),
    "Age":            ages,
    "YearsOfService": years_service,
    "CurrentSalary":  current_salary.round(0),
})

# Only vested employees count
df = df[df["YearsOfService"] >= VESTING_YEARS].copy()

# ============================================================
# 3. PROJECT FINAL SALARY & ACCRUED BENEFIT
# ============================================================
df["YearsToRetirement"] = RETIREMENT_AGE - df["Age"]
df["FinalSalary"]       = df["CurrentSalary"] * (1 + SALARY_GROWTH) ** df["YearsToRetirement"]
df["AccruedBenefit"]    = BENEFIT_FACTOR * df["YearsOfService"] * df["FinalSalary"]

# ============================================================
# 4. MORTALITY ADJUSTMENT (Demographic Assumption)
# ============================================================
# Simplified exponential survival: probability of surviving to retirement
df["SurvivalProb"]      = np.exp(-MORTALITY_RATE * df["YearsToRetirement"])
df["AccruedBenefit_Adj"] = df["AccruedBenefit"] * df["SurvivalProb"]

# ============================================================
# 5. LIABILITY CALCULATION FUNCTION
# ============================================================
def calc_liability(data, discount_rate, use_mortality=True):
    d = data.copy()
    d["DiscountFactor"] = 1 / (1 + discount_rate) ** d["YearsToRetirement"]
    benefit_col = "AccruedBenefit_Adj" if use_mortality else "AccruedBenefit"
    d["PV_Benefit"] = d[benefit_col] * d["DiscountFactor"]
    return d

# Base case
base = calc_liability(df, DISCOUNT_RATE)
total_liability = base["PV_Benefit"].sum()

# Without mortality (for comparison)
base_no_mort = calc_liability(df, DISCOUNT_RATE, use_mortality=False)
total_liability_no_mort = base_no_mort["PV_Benefit"].sum()

# ============================================================
# 6. SENSITIVITY ANALYSIS (Discount Rate)
# ============================================================
discount_rates = [0.05, 0.06, 0.07, 0.08, 0.09]
liabilities = [calc_liability(df, dr)["PV_Benefit"].sum() for dr in discount_rates]

sensitivity = pd.DataFrame({
    "DiscountRate":   [f"{dr:.0%}" for dr in discount_rates],
    "TotalLiability": liabilities,
    "ChangeFromBase": [l - total_liability for l in liabilities],
})

# ============================================================
# 7. SENSITIVITY ANALYSIS (Salary Growth)
# ============================================================
salary_growth_rates = [0.04, 0.05, 0.06, 0.07, 0.08]
salary_liabilities = []
for sg in salary_growth_rates:
    temp = df.copy()
    temp["FinalSalary"]      = temp["CurrentSalary"] * (1 + sg) ** temp["YearsToRetirement"]
    temp["AccruedBenefit"]   = BENEFIT_FACTOR * temp["YearsOfService"] * temp["FinalSalary"]
    temp["AccruedBenefit_Adj"] = temp["AccruedBenefit"] * temp["SurvivalProb"]
    salary_liabilities.append(calc_liability(temp, DISCOUNT_RATE)["PV_Benefit"].sum())

salary_sensitivity = pd.DataFrame({
    "SalaryGrowth":   [f"{sg:.0%}" for sg in salary_growth_rates],
    "TotalLiability": salary_liabilities,
    "ChangeFromBase": [l - total_liability for l in salary_liabilities],
})

# ============================================================
# 8. FUNDING STATUS
# ============================================================
funded_status = PLAN_ASSETS - total_liability
funding_ratio = PLAN_ASSETS / total_liability

# ============================================================
# 9. PRINT REPORT
# ============================================================
def fmt(x):
    return f"Rs {x:,.0f}"

print("=" * 70)
print("   PENSION LIABILITY VALUATION & SENSITIVITY ANALYSIS")
print("=" * 70)

print(f"\n[PLAN ASSUMPTIONS]")
print(f"  Benefit Formula        : {BENEFIT_FACTOR*100:.1f}% x Final Salary x Years of Service")
print(f"  Retirement Age         : {RETIREMENT_AGE}")
print(f"  Salary Growth          : {SALARY_GROWTH*100:.0f}%")
print(f"  Discount Rate (Base)   : {DISCOUNT_RATE*100:.0f}%")
print(f"  Vesting Period         : {VESTING_YEARS} years")
print(f"  Mortality Rate         : {MORTALITY_RATE*100:.1f}% annual (simplified)")
print(f"  Eligible Employees     : {len(df)}")

print(f"\n[LIABILITY - BASE CASE @ {DISCOUNT_RATE*100:.0f}%]")
print(f"  Total Pension Liability (with mortality)    : {fmt(total_liability)}")
print(f"  Total Pension Liability (no mortality)      : {fmt(total_liability_no_mort)}")
print(f"  Mortality impact on liability               : {fmt(total_liability_no_mort - total_liability)}")
print(f"  Plan Assets                                 : {fmt(PLAN_ASSETS)}")
print(f"  Funded Status                               : {fmt(funded_status)}")
print(f"  Funding Ratio                               : {funding_ratio*100:.2f}%")

print(f"\n[SENSITIVITY ANALYSIS - DISCOUNT RATE]")
print(sensitivity.to_string(index=False, float_format=lambda x: f"{x:,.0f}"))

print(f"\n[SENSITIVITY ANALYSIS - SALARY GROWTH]")
print(salary_sensitivity.to_string(index=False, float_format=lambda x: f"{x:,.0f}"))

print(f"\n[KEY INSIGHTS]")
print(f"  Discount rate 7% -> 6%: liability changes by {fmt(liabilities[1] - total_liability)}")
print(f"  Discount rate 7% -> 8%: liability changes by {fmt(liabilities[3] - total_liability)}")
print(f"  Salary growth 6% -> 5%: liability changes by {fmt(salary_liabilities[1] - total_liability)}")
print(f"  Salary growth 6% -> 7%: liability changes by {fmt(salary_liabilities[3] - total_liability)}")
print(f"  Liability range (5%-9% discount): {fmt(min(liabilities))} to {fmt(max(liabilities))}")

print(f"\n[DE-RISKING RECOMMENDATIONS]")
print("  1. Lump-Sum Cashout   - Offer terminated vested participants lump sum to exit")
print("  2. Plan Freeze        - Stop new benefit accruals for existing employees")
print("  3. Asset-Liability    - Match bond duration to liability duration")
print("  4. Plan Termination   - Transfer liability via insurer buy-out")

print("\n" + "=" * 70)

# ============================================================
# 10. SAVE OUTPUTS
# ============================================================
sensitivity.to_csv("sensitivity_discount_rate.csv", index=False)
salary_sensitivity.to_csv("sensitivity_salary_growth.csv", index=False)
base.to_csv("employee_liability_details.csv", index=False)

# --- Chart 1: Discount Rate Sensitivity ---
plt.figure(figsize=(8, 5))
colors = ["#d9534f" if l > total_liability else "#5cb85c" for l in liabilities]
plt.bar(sensitivity["DiscountRate"], liabilities, color=colors)
plt.axhline(y=total_liability, color="black", linestyle="--", linewidth=1,
            label=f"Base Case ({DISCOUNT_RATE:.0%})")
plt.xlabel("Discount Rate")
plt.ylabel("Total Pension Liability (Rs)")
plt.title("Sensitivity of Pension Liability to Discount Rate")
plt.legend()
plt.tight_layout()
plt.savefig("chart_sensitivity_discount.png", dpi=150)
plt.close()

# --- Chart 2: Salary Growth Sensitivity ---
plt.figure(figsize=(8, 5))
colors = ["#d9534f" if l > total_liability else "#5cb85c" for l in salary_liabilities]
plt.bar(salary_sensitivity["SalaryGrowth"], salary_liabilities, color=colors)
plt.axhline(y=total_liability, color="black", linestyle="--", linewidth=1,
            label=f"Base Case ({SALARY_GROWTH:.0%})")
plt.xlabel("Salary Growth Rate")
plt.ylabel("Total Pension Liability (Rs)")
plt.title("Sensitivity of Pension Liability to Salary Growth")
plt.legend()
plt.tight_layout()
plt.savefig("chart_sensitivity_salary.png", dpi=150)
plt.close()

# --- Chart 3: Liability by Age Group ---
base["AgeGroup"] = pd.cut(base["Age"], bins=[20, 30, 40, 50, 61],
                          labels=["25-30", "31-40", "41-50", "51-60"])
age_liab = base.groupby("AgeGroup", observed=True)["PV_Benefit"].sum()

plt.figure(figsize=(8, 5))
age_liab.plot(kind="bar", color="steelblue")
plt.xlabel("Age Group")
plt.ylabel("PV of Benefits (Rs)")
plt.title("Pension Liability by Age Group")
plt.xticks(rotation=0)
plt.tight_layout()
plt.savefig("chart_age_group.png", dpi=150)
plt.close()

# --- Chart 4: Mortality Impact ---
plt.figure(figsize=(6, 5))
plt.bar(["With Mortality", "Without Mortality"],
        [total_liability, total_liability_no_mort],
        color=["steelblue", "#d9534f"])
plt.ylabel("Total Liability (Rs)")
plt.title("Impact of Mortality Assumption on Liability")
plt.tight_layout()
plt.savefig("chart_mortality_impact.png", dpi=150)
plt.close()

print("\nFiles saved:")
print("  - sensitivity_discount_rate.csv")
print("  - sensitivity_salary_growth.csv")
print("  - employee_liability_details.csv")
print("  - chart_sensitivity_discount.png")
print("  - chart_sensitivity_salary.png")
print("  - chart_age_group.png")
print("  - chart_mortality_impact.png")
     
======================================================================
   PENSION LIABILITY VALUATION & SENSITIVITY ANALYSIS
======================================================================

[PLAN ASSUMPTIONS]
  Benefit Formula        : 1.5% x Final Salary x Years of Service
  Retirement Age         : 60
  Salary Growth          : 6%
  Discount Rate (Base)   : 7%
  Vesting Period         : 5 years
  Mortality Rate         : 0.5% annual (simplified)
  Eligible Employees     : 380

[LIABILITY - BASE CASE @ 7%]
  Total Pension Liability (with mortality)    : Rs 35,082,800
  Total Pension Liability (no mortality)      : Rs 37,469,861
  Mortality impact on liability               : Rs 2,387,061
  Plan Assets                                 : Rs 500,000,000
  Funded Status                               : Rs 464,917,200
  Funding Ratio                               : 1425.20%

[SENSITIVITY ANALYSIS - DISCOUNT RATE]
DiscountRate  TotalLiability  ChangeFromBase
          5%      45,474,517      10,391,717
          6%      39,767,524       4,684,724
          7%      35,082,800               0
          8%      31,208,138      -3,874,662
          9%      27,979,525      -7,103,275

[SENSITIVITY ANALYSIS - SALARY GROWTH]
SalaryGrowth  TotalLiability  ChangeFromBase
          4%      27,810,150      -7,272,650
          5%      31,141,061      -3,941,739
          6%      35,082,800               0
          7%      39,767,524       4,684,724
          8%      45,357,999      10,275,200

[KEY INSIGHTS]
  Discount rate 7% -> 6%: liability changes by Rs 4,684,724
  Discount rate 7% -> 8%: liability changes by Rs -3,874,662
  Salary growth 6% -> 5%: liability changes by Rs -3,941,739
  Salary growth 6% -> 7%: liability changes by Rs 4,684,724
  Liability range (5%-9% discount): Rs 27,979,525 to Rs 45,474,517

[DE-RISKING RECOMMENDATIONS]
  1. Lump-Sum Cashout   - Offer terminated vested participants lump sum to exit
  2. Plan Freeze        - Stop new benefit accruals for existing employees
  3. Asset-Liability    - Match bond duration to liability duration
  4. Plan Termination   - Transfer liability via insurer buy-out

======================================================================

Files saved:
  - sensitivity_discount_rate.csv
  - sensitivity_salary_growth.csv
  - employee_liability_details.csv
  - chart_sensitivity_discount.png
  - chart_sensitivity_salary.png
  - chart_age_group.png
  - chart_mortality_impact.png
