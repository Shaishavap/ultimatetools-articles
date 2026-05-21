# Building a Text Case Converter in React — 8 Transforms with Regex, From camelCase to SCREAMING_SNAKE_CASE

Text case conversion sounds trivial until you try to handle all the edge cases. Converting "hello world" to `camelCase` is easy. But what about converting `XMLParser` to `snake_case`? Or handling mixed input like `"My API-Key_Value"` correctly?

Here's how the [Text Case Converter](https://ultimatetools.io/tools/text-tools/case-converter/) at Ultimate Tools implements 8 case transforms in React using regex-based transformations.

---

## The 8 Transforms

The component exposes these functions, each accepting and updating a shared `text` state:

```typescript
const [text, setText] = useState('');
```

### 1. UPPER CASE / lower case

```typescript
const toUpperCase = () => setText(text.toUpperCase());
const toLowerCase = () => setText(text.toLowerCase());
```

Trivial, but worth including for completeness. JavaScript's built-in `toUpperCase()` handles Unicode correctly (e.g., `ß` → `SS` in German).

---

### 2. Title Case

```typescript
const toTitleCase = () => {
  setText(
    text.toLowerCase().split(' ').map(word =>
      word.charAt(0).toUpperCase() + word.slice(1)
    ).join(' ')
  );
};
```

This capitalizes the first letter of each space-separated word. Note: it doesn't handle articles (a, an, the) specially — for a basic converter, consistent behavior is better than trying to apply AP style rules.

---

### 3. Sentence Case

```typescript
const toSentenceCase = () => {
  setText(
    text.toLowerCase().replace(/(^\s*\w|[.!?]\s*\w)/g, c => c.toUpperCase())
  );
};
```

The regex `/(^\s*\w|[.!?]\s*\w)/g` matches:
- `^\s*\w` — the first word character at the start of the string (after optional whitespace)
- `[.!?]\s*\w` — the first word character after a sentence-ending punctuation mark

Both get uppercased. This gives "Hello world. This is a test." from "hello world. this is a test."

---

### 4. camelCase

```typescript
const toCamelCase = () => {
  setText(
    text
      .replace(/(?:^\w|[A-Z]|\b\w)/g, (word, index) =>
        index === 0 ? word.toLowerCase() : word.toUpperCase()
      )
      .replace(/\s+/g, '')
      .replace(/[-_]+/g, '')
  );
};
```

This works in three steps:
1. Capitalize the first letter of each word boundary (`\b\w`), but lowercase the very first character (`index === 0`)
2. Remove all whitespace
3. Remove hyphens and underscores

Input `"hello world"` → `"helloWorld"`
Input `"my-api-key"` → `"myApiKey"`

---

### 5. PascalCase

```typescript
const toPascalCase = () => {
  setText(
    text
      .replace(/[-_]+/g, ' ')           // normalize separators to spaces
      .replace(/[^\w\s]/g, '')           // remove non-word, non-space chars
      .replace(/\s+(.)(\w*)/g, ($1, $2, $3) =>
        `${$2.toUpperCase() + $3.toLowerCase()}`
      )
      .replace(/\w/, s => s.toUpperCase()) // capitalize the first char
      .replace(/\s+/g, '')
  );
};
```

PascalCase is camelCase but with the first letter also capitalized.

Input `"hello world"` → `"HelloWorld"`
Input `"my_api_key"` → `"MyApiKey"`

---

### 6. snake_case and kebab-case

These two share the same word-splitting regex:

```typescript
const toSnakeCase = () => {
  setText(
    text
      .match(/[A-Z]{2,}(?=[A-Z][a-z]+[0-9]*|\b)|[A-Z]?[a-z]+[0-9]*|[A-Z]|[0-9]+/g)
      ?.map(x => x.toLowerCase())
      .join('_') || text
  );
};

const toKebabCase = () => {
  setText(
    text
      .match(/[A-Z]{2,}(?=[A-Z][a-z]+[0-9]*|\b)|[A-Z]?[a-z]+[0-9]*|[A-Z]|[0-9]+/g)
      ?.map(x => x.toLowerCase())
      .join('-') || text
  );
};
```

The regex is the interesting part:

```
/[A-Z]{2,}(?=[A-Z][a-z]+[0-9]*|\b)|[A-Z]?[a-z]+[0-9]*|[A-Z]|[0-9]+/g
```

Breaking it down:
- `[A-Z]{2,}(?=[A-Z][a-z]+[0-9]*|\b)` — a run of uppercase letters followed by a lookahead for an uppercase+lowercase sequence (handles `XMLParser` → `XML`, `Parser`)
- `[A-Z]?[a-z]+[0-9]*` — an optional uppercase followed by lowercase letters (handles `Hello`)
- `[A-Z]` — a lone uppercase letter
- `[0-9]+` — a run of digits

This splits `"XMLParser"` correctly into `["XML", "Parser"]` → `xml_parser`, rather than the naive approach that would give `x_m_l_parser`.

Input `"helloWorld"` → `["hello", "World"]` → `hello_world`
Input `"XMLParser"` → `["XML", "Parser"]` → `xml_parser`
Input `"myAPIKey"` → `["my", "API", "Key"]` → `my_api_key`

---

### 7. Inverse Case

```typescript
const toInverseCase = () => {
  setText(
    text.split('').map(char =>
      char === char.toUpperCase() ? char.toLowerCase() : char.toUpperCase()
    ).join('')
  );
};
```

Flips each character's case. Uppercase becomes lowercase and vice versa. `"Hello World"` → `"hELLO wORLD"`.

---

### 8. Alternating Case

```typescript
const toAlternatingCase = () => {
  setText(
    text.toLowerCase().split('').map((char, index) =>
      index % 2 === 0 ? char.toLowerCase() : char.toUpperCase()
    ).join('')
  );
};
```

Alternates lowercase and uppercase by character index. `"hello world"` → `"hElLo wOrLd"`. Starts by lowercasing everything, then applies the alternating pattern — so the result is predictable regardless of input case.

---

## The UI Pattern

The component uses a single shared textarea with a button grid below it. Each button calls its transform function, which calls `setText()` with the result:

```tsx
<textarea
  value={text}
  onChange={e => setText(e.target.value)}
  placeholder="Type or paste your text here..."
  className="w-full h-48 rounded-xl border ..."
/>

<div className="grid grid-cols-2 gap-2 sm:grid-cols-4">
  <button onClick={toUpperCase}>UPPER CASE</button>
  <button onClick={toLowerCase}>lower case</button>
  <button onClick={toTitleCase}>Title Case</button>
  <button onClick={toSentenceCase}>Sentence case</button>
  <button onClick={toCamelCase}>camelCase</button>
  <button onClick={toPascalCase}>PascalCase</button>
  <button onClick={toSnakeCase}>snake_case</button>
  <button onClick={toKebabCase}>kebab-case</button>
  <button onClick={toInverseCase}>iNVERSE cASE</button>
  <button onClick={toAlternatingCase}>aLtErNaTiNg</button>
</div>
```

The transforms are destructive — they replace the text in place. There's no "before" state preserved. A copy-to-clipboard button lets users grab the result after each transform.

---

## Edge Cases Worth Knowing

**Numbers in identifiers** — the snake/kebab regex handles `myVar2Name` correctly, producing `my_var_2_name`. Not all converters do.

**Non-ASCII text** — `toUpperCase()` and `toLowerCase()` handle Unicode. The regex-based transforms (camel, pascal, snake, kebab) are ASCII-focused. Passing Arabic or CJK text through the word-splitting regex will likely return empty or the original text.

**Empty string** — the snake/kebab transform uses `?.map(...) || text` as a fallback if `match()` returns `null`, which happens on empty input.

---

## Summary

| Case | Key technique |
|---|---|
| Title | Split on space, capitalize first char per word |
| Sentence | Regex match after `.`, `!`, `?` |
| camelCase | `\b\w` boundary match, lowercase first |
| PascalCase | Same as camelCase, uppercase first |
| snake_case | Word-aware regex + `.join('_')` |
| kebab-case | Same regex + `.join('-')` |
| Inverse | Char-by-char case flip |
| Alternating | Even/odd index case toggle |

The [Text Case Converter](https://ultimatetools.io/tools/text-tools/case-converter/) is live at Ultimate Tools — paste any text and convert between 8 cases instantly.