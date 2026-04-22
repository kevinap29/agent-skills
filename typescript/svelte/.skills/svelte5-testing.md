# Svelte 5 Testing Skill

## Description

Expert skill for testing Svelte 5 applications using Vitest and Playwright. Covers unit testing Runes in `.svelte.ts` files, component testing with Svelte Testing Library, and end-to-end (E2E) testing. Use this skill when the user wants to ensure their logic is correct, prevent regressions, or setup a testing environment for Svelte 5.

## Installation

```bash
# Vitest (Unit & Component)
npm i -D vitest @testing-library/svelte jsdom

# Playwright (E2E)
npm init playwright@latest
```

## Unit Testing Runes (`.svelte.ts`)

Testing Runes is easy because they are just JavaScript/TypeScript.

```ts
// src/lib/counter.test.ts
import { describe, it, expect } from 'vitest';
import { Counter } from './counter.svelte';

describe('Counter', () => {
  it('should increment the count', () => {
    const counter = new Counter();
    expect(counter.count).toBe(0);
    
    counter.increment();
    expect(counter.count).toBe(1);
  });

  it('should compute double the count', () => {
    const counter = new Counter();
    counter.count = 5;
    expect(counter.double).toBe(10);
  });
});
```

## Component Testing (Svelte 5)

```ts
// src/components/Button.test.ts
import { render, screen, fireEvent } from '@testing-library/svelte';
import Button from './Button.svelte';
import { describe, it, expect, vi } from 'vitest';

describe('Button', () => {
  it('should render the label', () => {
    render(Button, { label: 'Click Me' });
    expect(screen.getByText('Click Me')).toBeTruthy();
  });

  it('should call the onclick handler', async () => {
    const handler = vi.fn();
    render(Button, { label: 'Click', onclick: handler });
    
    await fireEvent.click(screen.getByText('Click'));
    expect(handler).toHaveBeenCalled();
  });
});
```

## E2E Testing (Playwright)

```ts
// tests/login.spec.ts
import { test, expect } from '@playwright/test';

test('user can login successfully', async ({ page }) => {
  await page.goto('/login');
  
  await page.fill('input[name="email"]', 'user@example.com');
  await page.fill('input[name="password"]', 'password123');
  await page.click('button[type="submit"]');
  
  await expect(page).toHaveURL('/dashboard');
  await expect(page.locator('h1')).toContainText('Welcome');
});
```

## Testing Tips

- **Rune Reactivity in Tests**: Runes work automatically in Vitest as long as you are using the Svelte compiler/vite plugin.
- **`$effect` in Tests**: Effects might need a `flushSync` or a small delay to trigger in unit tests if they depend on microtasks.
- **Mocking `$app`**: When testing components that use SvelteKit modules like `$app/navigation`, use `vi.mock` to provide stubs.
- **Accessibility**: Use `vitest-axe` to run automated accessibility checks on your components during testing.
- **Playwright Trace**: Use `npx playwright show-trace` to debug failing E2E tests with a full visual replay.
