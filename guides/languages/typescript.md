# TypeScript Style Guide

> [Doctrine](../../README.md) > [Languages](../README.md) > TypeScript

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.html).

Extends [Google TypeScript Style Guide](google/typescript.html).

## Quick Reference

| Task | Tool | Command |
|------|------|---------|
| Lint | Biome[^1] | `biome check .` |
| Format | Biome[^1] | `biome format --write .` |
| Type check | tsc[^2] | `tsc --noEmit` |
| Semantic | - | - |
| Dead code | ts-prune[^3] | `ts-prune` |
| Coverage | c8[^4] | `c8 vitest` |
| Complexity | - | via Biome[^1] |
| Fuzz | - | - |
| Test perf | Vitest[^5] | `vitest --pool=threads` |

## Linting & Formatting: Biome

Projects **MUST** use Biome[^1] for linting and formatting unless extensive plugin ecosystems are required (see ESLint Alternative below).

### Why Biome

- **Performance**: 20x faster than ESLint[^6]+Prettier[^7]
- **Unified tooling**: Combines linting, formatting, and import organization in one tool
- **Zero config**: Works out of the box with sensible defaults
- **Better errors**: Provides more actionable error messages
- **Native binary**: No JavaScript runtime overhead

```bash
# Install
npm install --save-dev @biomejs/biome

# Initialize config
npx biome init

# Lint and format
npx biome check --write .

# CI mode
npx biome ci .
```

### Configuration (biome.json)

```json
{
  "$schema": "https://biomejs.dev/schemas/1.9.4/schema.json",
  "organizeImports": {
    "enabled": true
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "complexity": {
        "noExcessiveCognitiveComplexity": {
          "level": "warn",
          "options": { "maxAllowedComplexity": 15 }
        }
      },
      "suspicious": {
        "noExplicitAny": "error"
      },
      "style": {
        "useConst": "error",
        "noNonNullAssertion": "warn"
      }
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "semicolons": "always"
    }
  }
}
```

### ESLint Alternative

Projects **MAY** use ESLint[^6] for projects requiring extensive plugin ecosystems:

```bash
npm install --save-dev eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

```json
{
  "parser": "@typescript-eslint/parser",
  "plugins": ["@typescript-eslint"],
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/strict-type-checked",
    "plugin:@typescript-eslint/stylistic-type-checked"
  ],
  "parserOptions": {
    "project": true
  }
}
```

## Type Checking: tsc

Projects **MUST** enable strict type checking with TypeScript[^2]'s compiler.

### Why tsc

- **Official compiler**: The authoritative TypeScript type checker
- **Strict mode**: Catches entire classes of bugs at compile time
- **IDE integration**: Provides real-time type checking in editors
- **Zero-cost abstraction**: Type checking happens at build time with no runtime overhead

```bash
# Check types without emitting
npx tsc --noEmit

# Watch mode
npx tsc --noEmit --watch
```

### tsconfig.json (Strict)

Projects **MUST** enable all strict type checking options:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "skipLibCheck": true
  }
}
```

## Dead Code Detection: ts-prune

Projects **SHOULD** regularly check for unused exports using ts-prune[^3].

```bash
# Install
npm install --save-dev ts-prune

# Find unused exports
npx ts-prune
```

## Code Coverage: c8

Projects **SHOULD** measure code coverage using c8[^4] or Vitest[^5]'s built-in coverage provider.

c8[^4] uses V8's built-in coverage for fast, accurate results.

```bash
# With Vitest
npx c8 vitest run

# Threshold
npx c8 --check-coverage --lines 80 vitest run
```

Or with Vitest[^5]'s built-in coverage:

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      thresholds: {
        lines: 80,
        branches: 80,
      },
    },
  },
});
```

## Test Performance: Vitest

Projects **MUST** use Vitest[^5] as the test runner for TypeScript projects.

### Why Vitest

- **Native ESM support**: Works seamlessly with modern TypeScript modules
- **Vite-powered**: Leverages Vite[^8]'s transform pipeline for instant hot module replacement
- **Jest-compatible API**: Easy migration from Jest[^9] with familiar syntax
- **Built-in coverage**: Native V8 coverage support without additional configuration
- **Fast watch mode**: Near-instant test re-runs on file changes

```bash
# Install
npm install --save-dev vitest

# Run
npx vitest

# Parallel (threads)
npx vitest --pool=threads

# Watch mode
npx vitest --watch
```

### Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    pool: 'threads',
    poolOptions: {
      threads: {
        minThreads: 1,
        maxThreads: 4,
      },
    },
  },
});
```

## Pre-commit Configuration

```yaml
repos:
  - repo: https://github.com/biomejs/pre-commit
    rev: v0.6.0
    hooks:
      - id: biome-check
        additional_dependencies: ['@biomejs/biome@1.9.4']
```

## CI Pipeline

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - run: npx biome ci .
      - run: npx tsc --noEmit

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - run: npx vitest run --coverage
```

## Dependencies & Package Management

Projects **SHOULD** prefer pnpm[^10] for package management due to its speed and disk efficiency:

```bash
# Prefer pnpm for speed
pnpm install
pnpm add express
pnpm add -D @types/express
```

**Lock files**: Projects **MUST** commit lock files (`package-lock.json` or `pnpm-lock.yaml`) for reproducible builds.

**Version constraints**:
- `^1.2.3` - Compatible updates (>=1.2.3 <2.0.0)
- `~1.2.3` - Patch updates only (>=1.2.3 <1.3.0)
- `1.2.3` - Exact version (pinned)

**Vulnerability scanning**: Projects **MUST** regularly scan for vulnerabilities:

```bash
npm audit
pnpm audit
```

**Dependabot**: Projects **SHOULD** enable automated dependency updates[^11] (.github/dependabot.yml):

```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
```

## E2E & Acceptance Testing

Projects **SHOULD** use Playwright[^12] for browser E2E testing and **MAY** use Cucumber.js[^13] for BDD-style acceptance testing.

**Playwright[^12]** for browser E2E:

```typescript
import { test, expect } from '@playwright/test';

test('user login flow', async ({ page }) => {
  await page.goto('/login');
  await page.fill('#email', 'user@example.com');
  await page.fill('#password', 'password123');
  await page.click('button[type="submit"]');
  await expect(page).toHaveURL('/dashboard');
});
```

**Cucumber.js[^13]** for BDD acceptance:

```gherkin
Feature: User Authentication
  Scenario: Successful login
    Given I am on the login page
    When I enter valid credentials
    Then I should see the dashboard
```

```typescript
import { Given, When, Then } from '@cucumber/cucumber';

Given('I am on the login page', async function () {
  await this.page.goto('/login');
});
```

**Page Object Pattern**: Projects **SHOULD** use the Page Object Pattern for maintainable E2E tests:

```typescript
class LoginPage {
  constructor(private page: Page) {}

  async login(email: string, password: string) {
    await this.page.fill('#email', email);
    await this.page.fill('#password', password);
    await this.page.click('button[type="submit"]');
  }
}
```

## Thread Safety Testing

**Web Workers**:

```typescript
test('worker communication', async () => {
  const worker = new Worker('./worker.ts');
  const result = await new Promise((resolve) => {
    worker.onmessage = (e) => resolve(e.data);
    worker.postMessage({ type: 'process', data: [1, 2, 3] });
  });
  expect(result).toEqual([2, 4, 6]);
});
```

**SharedArrayBuffer and Atomics**:

```typescript
test('atomic operations', () => {
  const sab = new SharedArrayBuffer(4);
  const view = new Int32Array(sab);
  Atomics.store(view, 0, 42);
  expect(Atomics.load(view, 0)).toBe(42);
});
```

**Testing async race conditions**:

```typescript
test('no race condition in concurrent updates', async () => {
  const store = new ConcurrentStore();
  await Promise.all([
    store.increment('counter'),
    store.increment('counter'),
    store.increment('counter'),
  ]);
  expect(store.get('counter')).toBe(3);
});
```

## Idempotence Testing

**Retry-safe API calls**:

```typescript
test('duplicate requests produce same result', async () => {
  const result1 = await api.createOrder({ idempotencyKey: 'key-123', items: [] });
  const result2 = await api.createOrder({ idempotencyKey: 'key-123', items: [] });
  expect(result1.id).toBe(result2.id);
});
```

**Optimistic update testing**:

```typescript
test('rollback on conflict', async () => {
  const ui = renderComponent();
  ui.updateItem(1, { name: 'New Name' }); // Optimistic
  await waitFor(() => expect(ui.getItem(1).name).toBe('Original Name')); // Rolled back
});
```

## Reliability & Resilience Testing

**AbortController**:

```typescript
test('cancellable fetch', async () => {
  const controller = new AbortController();
  const promise = fetch('/api/data', { signal: controller.signal });
  controller.abort();
  await expect(promise).rejects.toThrow('aborted');
});
```

**Timeout and retry**:

```typescript
test('retries on failure', async () => {
  let attempts = 0;
  const fn = async () => {
    attempts++;
    if (attempts < 3) throw new Error('fail');
    return 'success';
  };
  const result = await retry(fn, { maxAttempts: 3 });
  expect(result).toBe('success');
  expect(attempts).toBe(3);
});
```

**Network error simulation**:

```typescript
test('handles network errors', async () => {
  server.use(
    http.get('/api/data', () => HttpResponse.networkError())
  );
  await expect(fetchData()).rejects.toThrow();
});
```

## Compatibility Testing

**Browser compatibility** (Playwright):

```typescript
import { chromium, firefox, webkit } from '@playwright/test';

for (const browserType of [chromium, firefox, webkit]) {
  test(`works in ${browserType.name()}`, async () => {
    const browser = await browserType.launch();
    const page = await browser.newPage();
    await page.goto('/');
    await expect(page.locator('h1')).toBeVisible();
    await browser.close();
  });
}
```

**Node.js version matrix** (.github/workflows/test.yml):

```yaml
strategy:
  matrix:
    node-version: [18, 20, 22]
steps:
  - uses: actions/setup-node@v4
    with:
      node-version: ${{ matrix.node-version }}
```

**tsconfig target options**:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM"]
  }
}
```

## Internationalization Testing

**UTF-8 handling** (native):

```typescript
test('handles unicode correctly', () => {
  const text = '你好世界 🌍';
  expect(text.length).toBe(6); // Code units
  expect([...text].length).toBe(6); // Code points
});
```

**i18next[^16] testing**:

```typescript
import i18n from 'i18next';

test('translates messages', async () => {
  await i18n.init({
    lng: 'es',
    resources: {
      es: { translation: { hello: 'Hola' } },
    },
  });
  expect(i18n.t('hello')).toBe('Hola');
});
```

**RTL layout testing**:

```typescript
test('supports RTL layout', async ({ page }) => {
  await page.goto('/?lang=ar');
  const dir = await page.locator('html').getAttribute('dir');
  expect(dir).toBe('rtl');
});
```

## Data Integrity Testing

Projects **MUST** validate runtime data using schema validation libraries like Zod[^14].

**Zod[^14] schema validation**:

```typescript
import { z } from 'zod';

const UserSchema = z.object({
  email: z.string().email(),
  age: z.number().min(0),
});

test('validates user data', () => {
  expect(() => UserSchema.parse({ email: 'invalid', age: -1 })).toThrow();
  expect(UserSchema.parse({ email: 'user@example.com', age: 25 })).toEqual({
    email: 'user@example.com',
    age: 25,
  });
});
```

**API contract testing**:

```typescript
test('API response matches schema', async () => {
  const response = await fetch('/api/users/1');
  const data = await response.json();
  expect(() => UserSchema.parse(data)).not.toThrow();
});
```

## A/B Testing & Feature Flags

**Feature flag libraries** (Unleash[^15]):

```typescript
import { Unleash } from 'unleash-client';

const unleash = new Unleash({
  url: 'https://unleash.example.com/api',
  appName: 'my-app',
});

if (unleash.isEnabled('new-checkout-flow')) {
  // New implementation
}
```

**Testing both flag states**:

```typescript
test.each([
  { flag: true, expected: 'New UI' },
  { flag: false, expected: 'Old UI' },
])('renders correct UI when flag is $flag', ({ flag, expected }) => {
  vi.spyOn(featureFlags, 'isEnabled').mockReturnValue(flag);
  const ui = render(<Component />);
  expect(ui.getByText(expected)).toBeInTheDocument();
});
```

## References

[^1]: [Biome](https://biomejs.dev/) - Fast formatter and linter for JavaScript/TypeScript
[^2]: [TypeScript](https://www.typescriptlang.org/) - TypeScript official website and documentation
[^3]: [ts-prune](https://github.com/nadeesha/ts-prune) - Find unused exports in TypeScript projects
[^4]: [c8](https://github.com/bcoe/c8) - Native V8 code coverage tool
[^5]: [Vitest](https://vitest.dev/) - Next generation testing framework powered by Vite
[^6]: [ESLint](https://eslint.org/) - Pluggable linting utility for JavaScript and TypeScript
[^7]: [Prettier](https://prettier.io/) - Opinionated code formatter
[^8]: [Vite](https://vitejs.dev/) - Next generation frontend tooling
[^9]: [Jest](https://jestjs.io/) - Delightful JavaScript testing framework
[^10]: [pnpm](https://pnpm.io/) - Fast, disk space efficient package manager
[^11]: [Dependabot](https://docs.github.com/en/code-security/dependabot) - Automated dependency updates
[^12]: [Playwright](https://playwright.dev/) - End-to-end testing for modern web apps
[^13]: [Cucumber.js](https://github.com/cucumber/cucumber-js) - BDD testing framework for JavaScript
[^14]: [Zod](https://zod.dev/) - TypeScript-first schema validation library
[^15]: [Unleash](https://www.getunleash.io/) - Open-source feature management platform
[^16]: [i18next](https://www.i18next.com/) - Internationalization framework for JavaScript

## See Also

- [Testing Guide](../testing.md) - Comprehensive testing strategies and best practices
- [CI Guide](../ci.md) - Continuous integration configuration and workflows
