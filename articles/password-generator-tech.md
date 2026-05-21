# Building a Cryptographically Secure Password Generator in React

Most password generator tutorials reach for `Math.random()`. It's easy, it works, and the output looks random. But `Math.random()` is a pseudo-random number generator — seeded deterministically, not suitable for security-sensitive output. For a password generator, that matters.

Here's how we built the one in [Ultimate Tools](https://ultimatetools.io/tools/utility-tools/password-generator/) using `crypto.getRandomValues()`, live entropy calculation, and a reactive React UI that regenerates on every settings change.

---

## Why Not Math.random()

`Math.random()` produces numbers from a deterministic algorithm. Given the seed and the algorithm, the output is predictable. Browsers don't expose the seed, but the underlying PRNG is not designed to be cryptographically unpredictable.

`crypto.getRandomValues()` is backed by the OS's cryptographically secure random source — the same source used for TLS keys, session tokens, and UUID generation. For passwords, always use this.

---

## Generating the Password

```ts
const generatePassword = () => {
    let charset = "";
    if (useUppercase) charset += "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
    if (useLowercase) charset += "abcdefghijklmnopqrstuvwxyz";
    if (useNumbers)   charset += "0123456789";
    if (useSymbols)   charset += "!@#$%^&*()_+~`|}{[]:;?><,./-=";

    if (charset === "") return;

    const randomValues = new Uint32Array(length);
    crypto.getRandomValues(randomValues);

    let retVal = "";
    for (let i = 0; i < length; i++) {
        retVal += charset.charAt(randomValues[i] % charset.length);
    }

    setPassword(retVal);
};
```

`crypto.getRandomValues()` fills a typed array with cryptographically random integers. We use `Uint32Array` — 32-bit unsigned integers — then map each value to a charset position with modulo.

**On modulo bias:** when `charset.length` doesn't evenly divide `2^32`, some characters appear slightly more often. For our charset sizes (26–92 characters), the maximum bias is `charset.length / 2^32` — less than 0.0000022%. Negligible for password generation. If you're building a cryptographic key derivation tool, use rejection sampling instead.

---

## Entropy Calculation

Entropy tells you how hard a password is to brute-force — in bits. The formula:

```
entropy = length × log₂(poolSize)
```

Where `poolSize` is the total number of possible characters.

```ts
const checkStrength = (pass: string) => {
    let poolSize = 0;
    if (useLowercase) poolSize += 26;
    if (useUppercase) poolSize += 26;
    if (useNumbers)   poolSize += 10;
    if (useSymbols)   poolSize += 30;

    if (poolSize === 0 || pass.length === 0) {
        setEntropy(0);
        setStrength(0);
        return;
    }

    const entropyVal = pass.length * Math.log2(poolSize);
    setEntropy(Math.round(entropyVal));

    if (entropyVal < 28)  setStrength(0); // Very Weak
    else if (entropyVal < 36)  setStrength(1); // Weak
    else if (entropyVal < 60)  setStrength(2); // Moderate
    else if (entropyVal < 128) setStrength(3); // Strong
    else                       setStrength(4); // Very Strong
};
```

**What the thresholds mean in practice:**

- **< 28 bits** — crackable instantly with modern hardware
- **28–36 bits** — crackable in seconds to minutes
- **36–60 bits** — hours to days for offline attacks
- **60–128 bits** — years with current hardware
- **128+ bits** — effectively uncrackable with foreseeable technology

A 16-character password with all four charsets has a pool of 92 characters: `16 × log₂(92) = 16 × 6.52 = 104 bits` — Strong. A 6-character lowercase-only password: `6 × log₂(26) = 6 × 4.7 = 28 bits` — Very Weak.

---

## Reactive Generation with useEffect

The password regenerates automatically whenever the user changes any setting:

```ts
useEffect(() => {
    generatePassword();
}, [length, useUppercase, useLowercase, useNumbers, useSymbols]);
```

No "Generate" button needed for the initial load or setting changes — the effect fires immediately. The user can still click the refresh icon to get a new password with the same settings.

One thing to watch: `generatePassword` is defined inside the component, so it's a new function reference on every render. The `useEffect` dependency array only contains the settings state values, not `generatePassword` itself — this is intentional. Adding `generatePassword` to the deps would cause an infinite loop (effect calls function → state updates → new function reference → effect fires again).

---

## Strength Meter UI

Four segments, each lighting up based on the current strength level:

```tsx
<div className="flex h-1.5 w-full gap-1 overflow-hidden rounded-full bg-zinc-200 dark:bg-zinc-700">
    {[0, 1, 2, 3].map((level) => (
        <div
            key={level}
            className={cn(
                "flex-1 transition-all duration-300",
                strength > level ? getStrengthColor() : "bg-transparent"
            )}
        />
    ))}
</div>
```

`strength > level` means: segment 0 lights up when strength > 0 (Weak or above), segment 1 when strength > 1 (Moderate or above), etc. All four light up only at Very Strong (strength = 4).

Color progression: red → orange → yellow → green → emerald, mapped via `getStrengthColor()`.

---

## Entropy Display

Showing entropy in bits alongside the strength label gives technically-minded users the actual number:

```tsx
<div className="flex justify-between text-xs font-medium uppercase text-zinc-500">
    <span>Strength: {getStrengthLabel()}</span>
    <span>Entropy: {entropy} bits</span>
</div>
```

At default settings (16 chars, all charsets): **104 bits — Strong**. Bump to 32 chars: **208 bits — Very Strong**. Drop to lowercase only, 8 chars: **37 bits — Moderate**.

---

## The Full Component State

```ts
const [password, setPassword]       = useState("");
const [length, setLength]           = useState(16);
const [useUppercase, setUseUppercase] = useState(true);
const [useLowercase, setUseLowercase] = useState(true);
const [useNumbers, setUseNumbers]   = useState(true);
const [useSymbols, setUseSymbols]   = useState(false);
const [copied, setCopied]           = useState(false);
const [strength, setStrength]       = useState(0);
const [entropy, setEntropy]         = useState(0);
```

Symbols default to off — many services don't accept all special characters, and users often get confused when a generated password is rejected. They can enable it explicitly.

---

## Summary

The key decisions:

- **`crypto.getRandomValues()`** over `Math.random()` — cryptographic randomness matters for passwords
- **`Uint32Array` + modulo** — simple and practical; bias is negligible at our charset sizes
- **Entropy in bits** — real metric, not a made-up score
- **`useEffect` on settings** — reactive regeneration without an explicit button
- **Symbols off by default** — reduces friction with services that restrict special characters

Try the live tool: [Password Generator](https://ultimatetools.io/tools/utility-tools/password-generator/)

This is part of [Ultimate Tools](https://ultimatetools.io/) — a free browser toolkit for PDFs, images, QR codes, and developer utilities.