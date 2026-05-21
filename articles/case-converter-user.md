# How to Convert Text Case Online — camelCase, snake_case, UPPER, and Title Case in One Click

Switching between cases is one of those micro-tasks that breaks flow constantly. You rename a variable, fix a heading, format a slug — and there you are, manually capitalizing and replacing spaces with underscores.

The [text case converter online](https://ultimatetools.io/tools/text-tools/case-converter/) handles ten case types in one click. Paste your text, click the conversion, copy the result.

## What It Converts

Ten case types, all in the same view:

| Button | Output example |
|---|---|
| UPPERCASE | HELLO WORLD |
| lowercase | hello world |
| Sentence case | Hello world. Second sentence. |
| Title Case | Hello World |
| camelCase | helloWorld |
| PascalCase | HelloWorld |
| snake_case | hello_world |
| kebab-case | hello-world |
| iNVERSE cASE | hELLO wORLD |
| aLtErNaTiNg cAsE | hElLo wOrLd |

## How to Use It

1. Open the [online case changer](https://ultimatetools.io/tools/text-tools/case-converter/)
2. Paste or type your text into the textarea
3. Click any conversion button — the text updates immediately in-place
4. Click another button to convert again from the current result
5. Copy with the clipboard button (top-right of the textarea)

The textarea stays editable throughout. You can type after converting, convert again, chain transformations. Nothing is sent to a server — it all runs in your browser.

---

## The Cases You'll Actually Use

**UPPERCASE / lowercase**

The simple ones. Useful for constants (`MAX_RETRIES`), database column names, or fixing text that was pasted in all caps.

**Title Case**

Each word's first letter capitalized. Used for article headings, product names, navigation labels. The tool applies it uniformly — no smart stop-word handling, every word gets capitalized.

**camelCase**

`helloWorld` — first word lowercase, every subsequent word starts with uppercase, no separators. Standard for JavaScript and TypeScript variable names, JSON keys, and React props.

**PascalCase**

`HelloWorld` — like camelCase but the first word is also capitalized. Used for class names in most OOP languages, React component names, TypeScript interfaces, and C# everything.

**snake_case**

`hello_world` — all lowercase, words joined with underscores. Standard in Python (variables, functions, file names), SQL column names, and Ruby. The most readable of the programmatic cases for long identifiers.

**kebab-case**

`hello-world` — same as snake but with hyphens. Standard for URL slugs, CSS class names, HTML `data-` attributes, and CLI flags. Cannot be used as a variable name in most languages (the hyphen is interpreted as subtraction).

---

## How camelCase and snake_case Detection Works

Converting free-text requires splitting the input into tokens first — not just on spaces, but also on existing case transitions and separators.

The [convert to camelcase online](https://ultimatetools.io/tools/text-tools/case-converter/) tool uses a regex that matches:
- Runs of uppercase letters followed by a lowercase word (`HTMLParser` → `HTML` and `Parser`)
- Mixed-case words starting mid-string
- Numbers as separate tokens

```
/[A-Z]{2,}(?=[A-Z][a-z]+[0-9]*|\b)|[A-Z]?[a-z]+[0-9]*|[A-Z]|[0-9]+/g
```

This is what drives snake_case and kebab-case too — only the join character differs (`_` vs `-`). It handles input that's already in camelCase, PascalCase, plain text with spaces, or mixed.

---

## Common Developer Use Cases

**Variable renaming:** Convert a display label to a code identifier. `User Full Name` → `userFullName` or `user_full_name`.

**Database column names:** Convert camelCase API fields to snake_case columns. `createdAt` → `created_at`.

**URL slugs:** Convert a page title to a URL-safe slug. `My Blog Post Title` → `my-blog-post-title`.

**CSS class names:** Convert a component name to a BEM-friendly class. `ButtonPrimary` → `button-primary`.

**Fixing pasted text:** Remove all caps from an email someone pasted in. One click.

---

The full [snake_case converter](https://ultimatetools.io/tools/text-tools/case-converter/) is live at Ultimate Tools — paste text, click a case, copy the result. No install, no login, nothing stored server-side.