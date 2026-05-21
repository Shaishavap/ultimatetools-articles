# EMI Formula, Reverse EMI, and Year-by-Year Amortization in JavaScript (With React Hooks)

EMI (Equated Monthly Instalment) is how most loan repayments work — a fixed payment every month for a fixed tenure that covers both principal and interest. The math behind it is compact, but implementing it correctly across three modes — standard EMI, reverse EMI (affordability), and amortization with prepayment — has a few tricky parts.

This covers the formulas, the edge cases, and a React implementation with hooks.

---

## The EMI Formula

```
EMI = P × r × (1+r)^n
      ─────────────────
        (1+r)^n − 1
```

Where:
- `P` = principal (loan amount)
- `r` = monthly interest rate = annualRate / 12 / 100
- `n` = tenure in months

```typescript
function calcEMI(principal: number, annualRate: number, tenureMonths: number): number {
  if (annualRate === 0) return principal / tenureMonths; // zero-interest edge case
  const r = annualRate / 12 / 100;
  return (principal * r * Math.pow(1 + r, tenureMonths)) / (Math.pow(1 + r, tenureMonths) - 1);
}
```

**Verification:** ₹500,000 at 8.5% for 20 years (240 months):
- r = 0.085 / 12 = 0.007083
- EMI ≈ ₹4,340/month
- Total payment ≈ ₹10.4L
- Total interest ≈ ₹5.4L

---

## Reverse EMI: How Much Can I Borrow?

Rearranging the EMI formula to solve for P given a fixed EMI budget:

```
P = EMI × [(1+r)^n − 1]
    ──────────────────────
         r × (1+r)^n
```

```typescript
function calcMaxLoan(emi: number, annualRate: number, tenureMonths: number): number {
  if (annualRate === 0) return emi * tenureMonths;
  const r = annualRate / 12 / 100;
  return (emi * (Math.pow(1 + r, tenureMonths) - 1)) / (r * Math.pow(1 + r, tenureMonths));
}
```

**Use case:** "I can afford ₹20,000/month. At 8.5% for 20 years, how much can I borrow?"

Answer: `calcMaxLoan(20000, 8.5, 240)` ≈ ₹23,03,000

This is genuinely useful and almost no free calculator offers it.

---

## Year-by-Year Amortization Table

An amortization table shows how each payment is split between principal and interest over the loan lifetime. For UI purposes, we aggregate by year (not month).

```typescript
interface AmortizationRow {
  year: number;
  openingBalance: number;
  principal: number;
  interest: number;
  closingBalance: number;
}

function buildAmortization(
  principal: number,
  annualRate: number,
  tenureMonths: number,
  extraMonthly: number
): AmortizationRow[] {
  const rows: AmortizationRow[] = [];
  let balance = principal;
  const r = annualRate / 12 / 100;
  const baseEmi = calcEMI(principal, annualRate, tenureMonths);
  let year = 1;
  let yearPrincipal = 0;
  let yearInterest = 0;

  for (let m = 1; m <= tenureMonths; m++) {
    const interest = balance * r;
    const principalPaid = Math.min(baseEmi - interest + extraMonthly, balance);
    balance -= principalPaid;
    yearPrincipal += principalPaid;
    yearInterest += interest;

    const isEndOfYear = m % 12 === 0 || m === tenureMonths || balance <= 0.01;
    if (isEndOfYear) {
      rows.push({
        year,
        openingBalance: rows.length === 0
          ? principal
          : rows[rows.length - 1].closingBalance + yearPrincipal,
        principal: yearPrincipal,
        interest: yearInterest,
        closingBalance: Math.max(balance, 0),
      });
      year++;
      yearPrincipal = 0;
      yearInterest = 0;
      if (balance <= 0.01) break;
    }
  }
  return rows;
}
```

The `extraMonthly` parameter handles prepayment — an additional amount paid on top of the base EMI each month. This reduces the balance faster, shortening the loan tenure and cutting total interest.

---

## Prepayment Impact

Given an extra monthly payment, how much interest does the borrower save and how many months earlier do they close?

```typescript
const amortRows = buildAmortization(P, r, n, extra);

// Total interest without prepayment
const noExtraInterest = totalInterest; // emi * n - P

// Total interest with prepayment
const withExtraInterest = amortRows.reduce((acc, row) => acc + row.interest, 0);

const savedInterest = noExtraInterest - withExtraInterest;

// Months saved = original tenure - actual months taken
// amortRows.length = years; each year covers up to 12 months
const monthsSaved = n - amortRows.length * 12;
```

> Note: `monthsSaved` is approximate — the last year in `amortRows` may have fewer than 12 months. For exact calculation, you'd track the actual month count instead of multiplying years × 12.

---

## React Component: State Design

```typescript
type Tab = "emi" | "reverse" | "compare";

export function EmiCalculator() {
  const [tab, setTab] = useState<Tab>("emi");
  const [currency, setCurrency] = useState<"₹" | "$">("₹");

  // EMI tab inputs
  const [loanAmount, setLoanAmount] = useState("500000");
  const [rate, setRate] = useState("8.5");
  const [tenure, setTenure] = useState("20");
  const [extraMonthly, setExtraMonthly] = useState("");

  // Reverse tab inputs
  const [desiredEMI, setDesiredEMI] = useState("20000");
  const [reverseRate, setReverseRate] = useState("8.5");
  const [reverseTenure, setReverseTenure] = useState("20");

  // Compare tab: second loan
  const [loan2Amount, setLoan2Amount] = useState("500000");
  const [loan2Rate, setLoan2Rate] = useState("9.5");
  const [loan2Tenure, setLoan2Tenure] = useState("20");

  // Region-auto-detected for news section
  const [currency, setCurrency] = useState<"₹" | "$">("₹");

  useEffect(() => {
    detectEmiRegion().then((region) => {
      if (region === "USA") setCurrency("$");
    });
  }, []);

  // All calculations are derived — no separate "calculate" button
  const P = parseFloat(loanAmount) || 0;
  const r = parseFloat(rate) || 0;
  const n = (parseFloat(tenure) || 0) * 12;
  const extra = parseFloat(extraMonthly) || 0;

  const emi = P > 0 && r > 0 && n > 0 ? calcEMI(P, r, n) : 0;
  const totalPayment = emi * n;
  const totalInterest = totalPayment - P;
  const amortRows = P > 0 && r > 0 && n > 0 ? buildAmortization(P, r, n, extra) : [];
```

Calculations are derived synchronously on every render — no `useEffect` for calculation, no stale results. Inputs change → results update immediately.

---

## Loan Comparison

The compare tab runs the same formulas on two loan configurations side by side:

```typescript
// Loan A
const emiA = calcEMI(parseFloat(loanAmount), parseFloat(rate), parseFloat(tenure) * 12);
const totalA = emiA * parseFloat(tenure) * 12;

// Loan B
const emiB = calcEMI(parseFloat(loan2Amount), parseFloat(loan2Rate), parseFloat(loan2Tenure) * 12);
const totalB = emiB * parseFloat(loan2Tenure) * 12;

const cheaper = totalA < totalB ? "A" : "B";
const savings = Math.abs(totalA - totalB);
```

Displaying "Loan B costs ₹X less overall" gives the user a direct answer instead of making them do the mental arithmetic.

---

## Currency Formatting for India

Indian number formatting uses lakhs and crores, not thousands and millions:

```typescript
function fmtINR(n: number): string {
  if (n >= 1_00_00_000) return `₹${(n / 1_00_00_000).toFixed(2)}Cr`;
  if (n >= 1_00_000)    return `₹${(n / 1_00_000).toFixed(2)}L`;
  return `₹${Math.round(n).toLocaleString("en-IN")}`;
}
```

`toLocaleString("en-IN")` formats with Indian grouping (12,34,567 instead of 1,234,567). The Cr/L shorthand keeps large numbers readable in result cards.

---

## The Tool

Live at [ultimatetools.io/tools/misc-tools/emi-calculator/](https://ultimatetools.io/tools/misc-tools/emi-calculator/).

Three tabs: standard EMI with amortization table, reverse EMI (affordability), and side-by-side loan comparison. Plus a region-aware loan news section that auto-detects India/USA/Global from your IP.

The formulas themselves are straightforward. The value is putting all three modes in one place, without ads gating the amortization table or requiring an account.