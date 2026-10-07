# @formfusion/licence-plates

Set of validation rules for worldwide licence plate numbers.

A zero-dependency lookup table of **78 country-specific regex patterns** for validating vehicle registration (licence plate) numbers. Every pattern works directly as an HTML [`pattern`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/pattern) attribute value, so you can use it with plain HTML, React, FormFusion, or `new RegExp()`.

## Why

Licence plate formats vary by country, and the separator alone is inconsistent. Germany is one to three letters, an optional hyphen or space, one to two letters, a space and up to four digits (`K-AB 1234`). France is `AB-123-CD`. Sweden is `ABC12A`. India is `MH12 AB 1234`. Brazil is `ABC-1234` — or `ABC 1234`. Every app with a vehicle field ends up re-implementing and re-maintaining this table.

This package ships it as one flat object so you don't have to.

## Installation

```bash
npm install @formfusion/licence-plates
```

```bash
yarn add @formfusion/licence-plates
```

## Usage

The package exports a single default object mapping lowercase ISO 3166-1 alpha-2 country codes to regex **strings**.

### ES modules

```js
import plates from '@formfusion/licence-plates';

console.log(plates.fr); // "^[A-Z]{2}-\\d{3}-[A-Z]{2}$"
```

### CommonJS

```js
const plates = require('@formfusion/licence-plates').default;

new RegExp(plates.fr).test('AB-123-CD'); // true
new RegExp(plates.fr).test('ab-123-cd'); // false - lowercase
new RegExp(plates.fr).test('AB 123 CD'); // false - wrong separator
```

### Plain HTML

The patterns are valid `pattern` attribute values, so they work without any JavaScript:

```html
<label for="plate">Licence plate (France)</label>
<input id="plate" name="plate" type="text" pattern="^[A-Z]{2}-\d{3}-[A-Z]{2}$" required />
```

### With FormFusion

FormFusion passes unknown `type` values straight through to the input's `pattern` attribute, so you can hand it a pattern directly:

```jsx
import React from 'react';
import { Form, Input } from 'formfusion';
import 'formfusion/style.css';
import plates from '@formfusion/licence-plates';

const MyForm = () => (
  <Form onSubmit={(data) => console.log('Submitted', data)}>
    <Input id="plate" name="plate" type={plates.de} label="Licence plate" required />
    <button type="submit">Submit</button>
  </Form>
);
```

Patterns compose with FormFusion's `rules` combinators if you need to accept more than one country:

```jsx
import { Input, rules } from 'formfusion';
import plates from '@formfusion/licence-plates';

// Accept either a German or an Austrian licence plate
<Input name="plate" type={rules.existIn([plates.de, plates.at])} />
```

### Dynamic country selection

```jsx
const [country, setCountry] = useState('de');

<Select name="country" value={country} onChange={setCountry}>
  {Object.keys(plates).map((code) => (
    <option key={code} value={code}>
      {code.toUpperCase()}
    </option>
  ))}
</Select>

{plates[country] ? (
  <Input name="plate" type={plates[country]} label="Licence plate" required />
) : (
  <Input name="plate" type="text" label="Licence plate" required />
)}
```

### Standalone validation

```js
import plates from '@formfusion/licence-plates';

export function isValidPlate(value, country) {
  const pattern = plates[String(country).toLowerCase()];

  if (!pattern) return false; // unknown country or no entry
  return new RegExp(pattern).test(value);
}

isValidPlate('K-AB 1234', 'DE'); // true
isValidPlate('AB-123-CD', 'FR'); // true
isValidPlate('ab-123-cd', 'FR'); // false
```

### TypeScript

Typings are hand-written in `index.d.ts` and mirror the lowercase keys via a mapped type. Because the declaration uses `export =`, you need `esModuleInterop` or `allowSyntheticDefaultImports`.

```ts
import plates from '@formfusion/licence-plates';

const fr: string = plates.fr;
// @ts-expect-error - unknown country
const xx: string = plates.xx;
```

## API

The export is a plain object with no functions or classes:

```ts
{ [countryCode: string]: string }
```

Country codes are **lowercase** (`de`, `fr`, `se`). Lookups are case-sensitive, so normalize user input first.

Coverage is 78 countries. There is no `gb`, `us`, `it`, `es`, `jp`, `kr`, `ru`, `tr`, `nl`, `pl`, `no`, `ch`, `ie`, `lu` or `ua` entry — check `Object.keys(plates)` before assuming a country is there.

`index.d.ts` declares all 78 keys, although `DE`, `AR`, `FI` and `BR` are each listed twice. The duplicates are harmless; TypeScript keeps the last one.

Enumerate the available codes at runtime with `Object.keys(plates)`.

## Caveats

Read these before relying on the patterns.

**`dk`, `dj`, `dm` and `do` can never match.** Their values are wrapped in literal slashes — the stored string is `/^[A-Z]{2}\d{4}$/` — so the regex demands a leading `/` and then a `^` that can never be satisfied. Treat those four as absent.

**Lowercase input is rejected by everything except `de`.** Most patterns are built from `[A-Z]`. `de` is the exception: it accepts lowercase letters and umlauts as well. Uppercase the value first unless you are deliberately matching `de`.

**`se` is very loose.** Its second branch, `[A-ZÅÄÖ ]{2,7}`, accepts any two to seven letters or spaces, so `ABCDEF` and `   ` both pass alongside a real `ABC12A`.

**Unknown keys are `undefined`, and `new RegExp(undefined)` matches everything.** `plates.gb` is not `null`, it is `undefined`, and the resulting regex is `(?:)`. Combined with the gaps in coverage, that means a British plate silently passes. Always guard before use.

**Format only.** There is no checksum and no region check. German plates carry a city prefix (`K` for Köln) inside the first letter group, but nothing validates it against a real district list, and Sweden's plate is not checked against the issued series.

## Development

```bash
git clone https://github.com/mitevskasara/formfusion-licence-plates.git
cd formfusion-licence-plates
npm install
npm run build
```

### How it works

All source lives in [`src/index.js`](src/index.js) as a single object of uppercase country codes. The last step lowercases every key before exporting, so `AT` becomes `at`.

[`esbuild.js`](esbuild.js) bundles that into a minified CommonJS `index.js` at the repo root, targeting Node 14. Consumers get the built file, so **changes are not live until you rebuild and commit `index.js`**:

```bash
npm run build
```

### Commit convention

This repo follows [Conventional Commits](https://www.conventionalcommits.org/), and `CHANGELOG.md` is generated from those subjects:

```
Feat: add Chilean plate pattern
Fix: drop literal slashes from Danish pattern
```

### Adding a country

1. Add the entry to `src/index.js`, using an uppercase country code.
2. Add the same uppercase code to the `LicencePlates` type in `index.d.ts`. The mapped type derives the lowercase key for you.
3. Run `npm run build` and commit the regenerated `index.js`.

## Related packages

Part of the FormFusion family of extracted validation rule sets:

- [`@formfusion/postcodes`](https://www.npmjs.com/package/@formfusion/postcodes)
- [`@formfusion/iban`](https://www.npmjs.com/package/@formfusion/iban)
- [`@formfusion/passports`](https://www.npmjs.com/package/@formfusion/passports)
- [`@formfusion/phones`](https://www.npmjs.com/package/@formfusion/phones)
- [`@formfusion/tin`](https://www.npmjs.com/package/@formfusion/tin)
- [`@formfusion/vat`](https://www.npmjs.com/package/@formfusion/vat)
- [`formfusion`](https://www.npmjs.com/package/formfusion) — the core library

## Issues

Report bugs and feature requests at https://github.com/mitevskasara/formfusion-licence-plates/issues.

## License

BSD-2-Clause. Copyright (c) 2023, Mitevska Sara.
