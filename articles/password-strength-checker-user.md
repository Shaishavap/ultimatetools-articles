# Password Strength Checker Free — Entropy Score, Crack Time, and Weakness Breakdown

Most "password strength" checkers are just traffic lights. Green if you have a symbol. Red if you don't. They don't tell you *why*, *how long* your password would survive an attack, or what specifically to fix.

The [Password Strength Checker](https://ultimatetools.io/tools/security-tools/password-generator/) tab in Ultimate Tools gives you the actual numbers: entropy bits, estimated crack time for both online and offline attacks, and a specific list of weaknesses.

---

## How Strength Is Actually Calculated

Strength is measured in **entropy bits** — a single number that captures how many guesses an attacker needs.

```
entropy = password_length × log₂(character_pool_size)
```

The pool grows with each character set you add:
- Lowercase only (a–z): pool = 26
- + Uppercase: pool = 52
- + Digits (0–9): pool = 62
- + Symbols (!@#$...): pool = ~94

A 12-character password using all four sets: `12 × log₂(94) ≈ 78 bits`

A vague "strong" badge tells you nothing. 78 bits tells you an offline attacker running 1 billion guesses per second would need 10 quintillion years.

---

## Online vs Offline Attack — Why Both Matter

The checker shows two crack times:

**Online attack (10 guesses/sec)** — Simulates a rate-limited login form. A 6-character lowercase password with 31 bits of entropy cracks in about 90 seconds online.

**Offline attack (1 billion/sec)** — Simulates an attacker who has stolen a hashed password database and is brute-forcing it on fast hardware. The same 6-character lowercase password cracks in 30 milliseconds offline.

This is why "don't reuse passwords" matters so much. If a site stores your password insecurely and gets breached, offline cracking is what happens next.

---

## What the Weakness List Catches

The checker flags specific issues:

- **Too short** — Under 8 characters fails most threat models
- **Short** — Under 12 characters is risky for sensitive accounts
- **Missing charsets** — Tells you specifically which set to add
- **Repeated characters** — `aaa`, `111` reduce effective entropy significantly
- **Common patterns** — Catches `123`, `qwerty`, `password`, `letmein`, dictionary words, and similar patterns that are cracked in seconds regardless of length

---

## How to Use It

1. Open the [password strength checker](https://ultimatetools.io/tools/security-tools/password-generator/)
2. Click the **Strength Checker** tab
3. Paste or type your existing password
4. Read the entropy score, strength label, crack times, and weakness list

Your password never leaves your browser — the analysis runs in JavaScript, no network requests.

---

## What "Strong" Actually Means in Numbers

| Entropy | Label | Offline crack time (1B/sec) |
|---|---|---|
| < 28 bits | Very Weak | Under 1 minute |
| 28–40 bits | Weak | Minutes to hours |
| 40–60 bits | Moderate | Days to months |
| 60–100 bits | Strong | Thousands of years |
| 100+ bits | Very Strong | Heat death of the universe |

For most accounts, aim for 60+ bits. For a password manager master password, 80+ bits.

---

Dashlane and 1Password lock strength analysis behind subscriptions. This checker is free with no account.

---

## Related Tools

- [free passphrase generator — 4 random words](https://ultimatetools.io/tools/security-tools/password-generator/) — if the checker flags your password as weak, switch to the Passphrase tab to generate a memorable multi-word replacement that scores higher on entropy
- [generate MD5 SHA-256 hash online free](https://ultimatetools.io/tools/coding-tools/hash-generator/) — see exactly how a password looks after hashing — understand what a breached password database actually contains, not the plaintext
- [free QR code generator for WiFi no watermark](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — encode your WiFi password as a QR code so guests never need to read it aloud or type it manually

---

**[Check your password strength for free →](https://ultimatetools.io/tools/security-tools/password-generator/)**