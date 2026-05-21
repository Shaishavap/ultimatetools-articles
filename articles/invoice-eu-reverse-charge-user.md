# EU VAT Reverse Charge Invoice — B2B Cross-Border Billing Free

When you sell services to a business in another EU country, you generally don't charge VAT — instead, the buyer accounts for it under the reverse charge mechanism. Getting this wrong means either overcharging your client (they paid VAT they shouldn't have) or undercharging (you owe VAT to your own tax authority). Here's how EU VAT reverse charge works, when it applies, and how to generate a compliant reverse charge invoice free.

## What Is EU VAT Reverse Charge?

Normally, the seller charges VAT and remits it to their tax authority. Under **reverse charge**, the buyer is responsible for accounting for the VAT — they report it on their own VAT return in their country, as both an output tax (as if they charged themselves) and an input tax (which they immediately reclaim if they're fully taxable). The net effect: zero VAT cash changes hands.

This mechanism exists because:
1. The EU doesn't want foreign businesses to register for VAT in every member state where they have customers
2. It simplifies cross-border B2B transactions
3. It reduces VAT fraud on intra-EU supplies

## When Does EU Reverse Charge Apply?

Reverse charge applies to **B2B (business-to-business) cross-border service supplies within the EU** when:

| Condition | Must be true |
|---|---|
| You (the supplier) are in an EU country | Yes |
| Your client is in a **different** EU country | Yes |
| Your client is **VAT-registered** in their country | Yes |
| The service falls under the **"general rule"** for where it's taxed | Yes (most services) |

Under the general rule (Article 44 of the EU VAT Directive), B2B services are taxed where the **customer** is established — not where you are. Since you're in a different country, you can't charge VAT there, so the buyer self-accounts via reverse charge.

### Services where reverse charge typically applies:
- Consulting, legal, accounting, and professional services
- Software development and SaaS
- Marketing and advertising
- Design and creative services
- IT support and managed services

### When reverse charge does NOT apply:
- **B2C (business-to-consumer)**: If your client is not VAT-registered, normal VAT rules apply — you may need to register for VAT in their country (or use the EU's OSS scheme)
- **Immovable property services**: Taxed where the property is located
- **In-person event services**: Taxed where the event takes place
- **Restaurant and catering**: Taxed where consumed

## What Must Be on a Reverse Charge Invoice?

A reverse charge invoice must include all standard invoice elements, plus:

| Field | Requirement |
|---|---|
| Your EU VAT number | e.g., DE123456789 (Germany), FR12345678901 (France) |
| Client's EU VAT number | Required to verify they're VAT-registered |
| Net amount (no VAT added) | VAT is 0% — the buyer accounts for it |
| Statutory reverse charge note | "VAT reverse charge applies" (or equivalent in local language) |
| Reference to the legal basis | Optional but recommended: "Art. 44 EU VAT Directive" or "Reverse charge — §13b UStG" (Germany) |

The mandatory note wording varies slightly by country:
- **English**: "Reverse charge — VAT to be accounted for by the recipient"
- **German**: "Steuerschuldnerschaft des Leistungsempfängers" (or simpler: "Reverse Charge")
- **French**: "Autoliquidation"
- **Spanish**: "Inversión del sujeto pasivo"

Most tax authorities accept "VAT reverse charge applies" in English for international B2B invoices.

## How to Generate an EU Reverse Charge Invoice Free

The [Invoice Generator](https://ultimatetools.io/tools/misc-tools/invoice-generator/) has a dedicated EU mode with a reverse charge toggle:

1. Open the invoice generator and click the **EU** flag
2. Enter your **EU VAT Number** (e.g., DE123456789) in the Tax ID field
3. Fill in your **business name and address**
4. Enter your **client's name, address, and their EU VAT Number**
5. Add your **line items** (description, quantity, rate — all ex-VAT)
6. Toggle **Reverse Charge** on — the VAT line removes from the total and the statutory note "VAT reverse charge applies" is added to the invoice
7. Add **payment terms** and payment details (IBAN + BIC for EU bank transfers)
8. Download the PDF

The downloaded PDF shows your VAT number, client's VAT number, 0% VAT, and the reverse charge note — everything needed for compliance.

## Verify Your Client's VAT Number

Before applying reverse charge, always **verify your client's EU VAT number** is valid. Use the EU's official VIES (VAT Information Exchange System) tool. An invalid VAT number means you can't apply reverse charge — you'd need to charge VAT at your local rate instead.

Keep a record of the VIES confirmation for each client — this is your evidence that reverse charge was correctly applied.

## Common EU Reverse Charge Mistakes

**1. Applying reverse charge to B2C clients**
If your client is an individual (not VAT-registered), reverse charge does not apply. You must charge VAT — either registering in their country or using the EU One Stop Shop (OSS) scheme.

**2. Missing the statutory note**
The invoice must state that the customer is responsible for VAT ("VAT reverse charge applies"). Without this note, the invoice isn't compliant and your client may face issues with their tax authority.

**3. Missing client's VAT number**
Your client's VAT number must appear on the invoice — it's the evidence that proves they're a VAT-registered business. Without it, your tax authority may challenge the reverse charge treatment.

**4. Not keeping VIES verification records**
If a client gives you a fake or expired VAT number and you apply reverse charge incorrectly, you may be liable for the VAT. Keep a timestamped VIES screenshot for each new client.

**5. Applying reverse charge within your own country**
Reverse charge for the general rule only applies to cross-border B2B supplies. If you're billing a VAT-registered business in your own country, you charge VAT normally (domestic reverse charge is a separate, country-specific rule that applies in specific sectors like construction).

## EU VAT Numbers by Country Format

| Country | Format | Example |
|---|---|---|
| Germany | DE + 9 digits | DE123456789 |
| France | FR + 2 chars + 9 digits | FR12345678901 |
| Spain | ES + letter/digit + 7 digits + letter/digit | ESX1234567X |
| Italy | IT + 11 digits | IT12345678901 |
| Netherlands | NL + 9 digits + B + 2 digits | NL123456789B01 |
| Poland | PL + 10 digits | PL1234567890 |
| Ireland | IE + 7 digits + 1-2 letters | IE1234567X |

## Related Tools

- [Invoice Generator](https://ultimatetools.io/tools/misc-tools/invoice-generator/) — EU VAT reverse charge invoice PDF, VAT number fields, statutory note auto-added
- [eSign PDF](https://ultimatetools.io/tools/pdf-tools/esign-pdf/) — digitally sign the invoice before sending
- [Compress PDF](https://ultimatetools.io/tools/pdf-tools/compress-pdf/) — reduce invoice PDF size for email
- [Merge PDF](https://ultimatetools.io/tools/pdf-tools/merge-pdf/) — combine invoice with contract or scope of work

---

Generate EU VAT reverse charge invoices free — VAT numbers, 0% VAT, statutory note auto-added: [Invoice Generator](https://ultimatetools.io/tools/misc-tools/invoice-generator/)