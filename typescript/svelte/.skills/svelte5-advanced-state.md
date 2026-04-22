# Svelte 5 Advanced State Skill

## Description

Expert skill for managing complex, shared, and persistent state in Svelte 5. Covers the Context API with Runes, reactive classes in `.svelte.ts` files, state synchronization across components, and local storage persistence. Use this skill when the user needs to share state globally, avoid prop drilling, or build complex stateful services.

## Reactive Classes (`.svelte.ts`)

The best way to encapsulate logic in Svelte 5 is using classes.

```ts
// src/lib/cart.svelte.ts
export class Cart {
  items = $state<{ id: string; qty: number }[]>([]);
  totalItems = $derived(this.items.reduce((acc, item) => acc + item.qty, 0));

  addItem(id: string) {
    const existing = this.items.find(i => i.id === id);
    if (existing) {
      existing.qty += 1;
    } else {
      this.items.push({ id, qty: 1 });
    }
  }

  removeItem(id: string) {
    this.items = this.items.filter(i => i.id !== id);
  }
}

// Singleton pattern
export const cart = new Cart();
```

## Context API with Runes

Use Context to avoid prop drilling for state that is "global" to a subtree.

```svelte
<!-- Root.svelte -->
<script lang="ts">
  import { setContext } from 'svelte';
  import { Cart } from '$lib/cart.svelte';

  const cart = new Cart();
  setContext('cart', cart);
</script>

<slot />

<!-- DeepChild.svelte -->
<script lang="ts">
  import { getContext } from 'svelte';
  import type { Cart } from '$lib/cart.svelte';

  const cart = getContext<Cart>('cart');
</script>

<button onclick={() => cart.addItem('123')}>
  Add to Cart ({cart.totalItems})
</button>
```

## Persistent State (Local Storage)

```ts
// src/lib/settings.svelte.ts
export class Settings {
  #theme = $state(localStorage.getItem('theme') ?? 'light');

  get theme() {
    return this.#theme;
  }

  set theme(value: string) {
    this.#theme = value;
    localStorage.setItem('theme', value);
  }

  toggle() {
    this.theme = this.theme === 'light' ? 'dark' : 'light';
  }
}
```

## Using Effects for Sync

```ts
$effect.root(() => {
  $effect(() => {
    console.log('Cart updated:', cart.items.length);
  });
});
```

## State Management Tips

- **Getters/Setters**: Use private fields (`#field`) with getters and setters in classes to control how state is modified.
- **`.svelte.ts`**: Always use the `.svelte.ts` (or `.svelte.js`) extension for files containing Runes outside of components.
- **Untracking**: Use `untrack(() => ...)` when you want to read a state inside an effect/derived without creating a dependency.
- **Context over Singletons**: Use the Context API if you need multiple instances of the same state in different parts of the app or if you are worried about SSR state leakage.
- **SSR Safety**: Be careful with `localStorage` or `window` in `.svelte.ts` files; wrap them in `if (browser)` checks.
