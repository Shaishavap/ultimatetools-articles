# Convert snake_case to camelCase Online Free — 8 Text Case Conversions for Developers in One Tool

You copy a variable name from a Python function — `user_first_name` — and need to paste it into a JavaScript object as `userFirstName`. Or you're normalizing API response keys to match your TypeScript interface. Or you're turning a database column name into a React prop.

Text case conversion is a frequent micro-task in cross-language and full-stack work. Here's a reference for when each format is used and how to convert between them instantly.

---

## The 8 Case Formats Developers Actually Use

**snake_case** — Python variables and functions, database column names, environment variable names. All lowercase, words joined by underscores. Constants use `SCREAMING_SNAKE_CASE` (all caps).

**camelCase** — JavaScript and TypeScript variables, JSON keys, Java and Kotlin methods. First word lowercase, each subsequent word capitalized. No separators.

**PascalCase** — Class names across most languages, React components, TypeScript interfaces and types, C# everything. Same as camelCase but the first word is also capitalized.

**kebab-case** — URL slugs, CSS class names, HTML data attributes, npm package names. Cannot be used as a variable name in most languages because the hyphen is parsed as a minus operator.

**SCREAMING_SNAKE_CASE** — Constants and environment variables. `API_BASE_URL`, `MAX_RETRY_COUNT`, `.env` file keys.

**Title Case** — Document headings, article titles, product names. "The Quick Brown Fox". Not used in code identifiers.

**lowercase / UPPERCASE** — Normalization before comparison, CSS text transforms, database IDs.

---

## When You Actually Need to Convert

**API response to TypeScript model** — External APIs commonly return `snake_case` keys (`created_at`, `user_id`, `is_active`). TypeScript interfaces use `camelCase`. Paste the full list of response keys, convert to camelCase in bulk, paste into your interface.

**Python backend to JavaScript frontend** — Python naming conventions are `snake_case`. JavaScript naming conventions are `camelCase`. Any shared data model needs conversion at the boundary.

**Database schema to ORM model** — Postgres and MySQL conventions use `snake_case` column names. ORMs often expect `camelCase` or `PascalCase` on the model class. Renaming one column at a time by hand introduces typos.

**URL slug generation** — A page title like "How to Convert PDF Pages" becomes `how-to-convert-pdf-pages` (kebab-case) for the URL. Title-to-kebab conversion in bulk is faster than typing each slug manually.

**Renaming across codebases** — Moving a module from one language to another means every identifier needs a naming convention change. Finding and replacing by hand misses things.

---

## Convert Online — All 8 Formats, Free

Paste any text, select the target case, get the converted output. Works on single words, full variable names, and multi-line lists.

[snake_case to camelCase converter — 8 case formats, bulk conversion, free, no account](https://ultimatetools.io/tools/text-tools/case-converter/)

---

## Quick Reference Table

| Input | camelCase | PascalCase | snake_case | kebab-case |
|---|---|---|---|---|
| `user first name` | `userFirstName` | `UserFirstName` | `user_first_name` | `user-first-name` |
| `api base url` | `apiBaseUrl` | `ApiBaseUrl` | `api_base_url` | `api-base-url` |
| `is active` | `isActive` | `IsActive` | `is_active` | `is-active` |
| `created at` | `createdAt` | `CreatedAt` | `created_at` | `created-at` |

---

## A Note on Bulk Conversion

The converter handles multi-line input — paste a full list of identifiers and all of them convert in one operation. Useful when you're adapting an API response schema or renaming a full set of database columns at once.

[text case converter for developers — bulk convert snake_case, camelCase, PascalCase, kebab-case](https://ultimatetools.io/tools/text-tools/case-converter/)