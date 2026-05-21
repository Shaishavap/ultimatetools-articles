# Building a BMI Calculator with Metric & Imperial Support in React

BMI calculators look deceptively simple — a couple of inputs and a formula. But once you add metric/imperial switching, a split ft+inches input, and live reactive updates, there are a few patterns worth thinking through. Here's how we built the one in [Ultimate Tools](https://ultimatetools.io/tools/health-tools/bmi-calculator/).

---

## The Two Formulas

BMI has different formulas depending on the unit system:

**Metric** — weight in kg, height in metres:
```
BMI = weight(kg) / height(m)²
```

**Imperial** — weight in lbs, height in inches:
```
BMI = 703 × weight(lbs) / height(in)²
```

The `703` constant is the conversion factor that reconciles the imperial units. Both formulas produce the same BMI value for the same person — it's just a unit conversion.

```ts
if (unit === "imperial") {
    const totalInches = h * 12 + i; // ft → total inches
    const bmiValue = (703 * w) / (totalInches * totalInches);
    setBmi(Math.round(bmiValue * 10) / 10);
} else {
    const hMeters = h / 100; // cm → metres
    const bmiValue = w / (hMeters * hMeters);
    setBmi(Math.round(bmiValue * 10) / 10);
}
```

`Math.round(bmiValue * 10) / 10` gives one decimal place without floating point noise — `22.4` instead of `22.400000000000002`.

---

## The Imperial Height Input: ft + inches

Imperial height isn't a single number — people say "5 feet 11 inches", not "71 inches". So we split the input into two fields:

```tsx
<div className="flex gap-2">
    <input
        type="number"
        value={height}
        onChange={(e) => setHeight(e.target.value)}
        placeholder="ft"
    />
    <input
        type="number"
        value={inches}
        onChange={(e) => setInches(e.target.value)}
        placeholder="in"
    />
</div>
```

`height` stores feet, `inches` stores the remainder. In the formula we combine them:

```ts
const totalInches = h * 12 + (parseFloat(inches) || 0);
```

The `|| 0` fallback handles the case where the inches field is empty — common when someone enters "5 ft" without adding inches yet. Without it, `parseFloat("")` returns `NaN`, which breaks the formula.

---

## Dual useEffect Pattern

The component uses two separate `useEffect` calls — one for BMI calculation, one for categorisation:

```ts
// Effect 1: recalculate BMI whenever inputs or unit changes
useEffect(() => {
    calculateBmi();
}, [height, inches, weight, unit]);

// Effect 2: update category whenever BMI changes
useEffect(() => {
    if (bmi === null) return;

    if (bmi < 18.5) setCategory("Underweight");
    else if (bmi < 25) setCategory("Normal weight");
    else if (bmi < 30) setCategory("Overweight");
    else setCategory("Obese");
}, [bmi]);
```

Why split them instead of doing both in one effect?

The categorisation logic only needs to run when `bmi` changes — not when the raw inputs change. Keeping them separate makes the data flow explicit: inputs → BMI value → category. Each effect has a clear single responsibility.

You could combine them, but the dependency array would grow (`[height, inches, weight, unit]`) even though categorisation doesn't directly depend on those values.

---

## BMI Categories

The WHO BMI classification used in the component:

| BMI Range | Category |
|---|---|
| < 18.5 | Underweight |
| 18.5 – 24.9 | Normal weight |
| 25 – 29.9 | Overweight |
| ≥ 30 | Obese |

Each category maps to a colour via `getCategoryColor()`:

```ts
const getCategoryColor = () => {
    if (category === "Underweight") return "bg-blue-100 text-blue-700 ...";
    if (category === "Normal weight") return "bg-green-100 text-green-700 ...";
    if (category === "Overweight") return "bg-yellow-100 text-yellow-700 ...";
    return "bg-red-100 text-red-700 ..."; // Obese
};
```

The result card's background and text colour update dynamically as the user types — no button press needed.

---

## Guarding Against Invalid Input

Both inputs accept `type="number"`, but users can still submit empty fields or zero. We guard at the top of `calculateBmi()`:

```ts
const h = parseFloat(height);
const w = parseFloat(weight);

if (isNaN(h) || isNaN(w) || h <= 0 || w <= 0) {
    setBmi(null);
    setCategory("");
    return;
}
```

`setBmi(null)` clears the result card and shows the placeholder text instead — cleaner than showing `NaN` or `0`.

The imperial path has an additional guard:

```ts
const totalInches = h * 12 + i;
if (totalInches <= 0) return;
```

A user could enter `0` feet and `0` inches, which would divide by zero. This guard prevents that without showing an error.

---

## Unit Switching State

When the user switches between Metric and Imperial, the input fields stay populated but the formula changes. We keep `height`, `inches`, and `weight` as raw string state — not parsed numbers — so the fields don't clear on toggle:

```ts
const [height, setHeight] = useState<string>("");
const [inches, setInches] = useState<string>("");  // imperial only
const [weight, setWeight] = useState<string>("");
```

Using `string` instead of `number` for controlled inputs avoids the annoying React behaviour where a `number` input clears itself when you type a decimal point mid-entry (`"70."` would be `70` as a number, triggering a re-render that removes the dot).

The unit toggle only renders the inches field for imperial:

```tsx
{unit === "metric" ? (
    <input value={height} ... placeholder="e.g. 175" />
) : (
    <div className="flex gap-2">
        <input value={height} ... placeholder="ft" />
        <input value={inches} ... placeholder="in" />
    </div>
)}
```

---

## Summary

The key decisions in this component:

- **Two formulas** — metric (`kg/m²`) and imperial (`703 × lbs/in²`) with a unit toggle
- **ft + inches split input** — `|| 0` fallback prevents NaN when inches field is empty
- **Dual `useEffect`** — inputs → BMI in one effect, BMI → category in another; clean separation of concerns
- **String state for inputs** — avoids decimal point clearing bug on controlled number inputs
- **Null BMI** — clears the result card cleanly instead of showing invalid output

Try the live tool: [BMI Calculator](https://ultimatetools.io/tools/health-tools/bmi-calculator/)

This is part of [Ultimate Tools](https://ultimatetools.io/) — a free browser toolkit for PDFs, images, QR codes, and developer utilities.