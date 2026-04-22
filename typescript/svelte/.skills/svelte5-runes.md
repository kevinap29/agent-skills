# Svelte 5 Runes Skill

## Description

Expert skill for Svelte 5 development using Runes (`$state`, `$derived`, `$effect`, `$props`, `$bindable`, `$inspect`), snippets, and the new component API. Use this skill whenever the user mentions Svelte 5, Runes, reactive state, or needs help refactoring Svelte 4 code to Svelte 5. This skill covers reactive state management, component communication, lifecycle handling, and performance optimization in Svelte 5 using TypeScript.

## Installation

```bash
# Create a new Svelte 5 project (using SvelteKit)
npx sv create my-app

# Select "SvelteKit demo app" or "Skeleton project"
# When prompted for Svelte version, choose 5 (Runes)
```

## Core Runes

### $state

Used to declare reactive state. Replaces `let` with reassignment.

```svelte
<script lang="ts">
  let count = $state(0);
  let user = $state({ name: 'John', age: 30 });

  function increment() {
    count += 1;
    user.age += 1; // Deep reactivity is automatic
  }
</script>

<button onclick={increment}>
  Count: {count} (Age: {user.age})
</button>
```

### $derived

Used for values that depend on other state. Replaces `$: double = count * 2`.

```svelte
<script lang="ts">
  let count = $state(0);
  let double = $derived(count * 2);
  let triple = $derived.by(() => {
    // For more complex logic
    return count * 3;
  });
</script>

<p>{count} x 2 = {double}</p>
<p>{count} x 3 = {triple}</p>
```

### $effect

Used for side effects. Replaces `$: { ... }` or `onMount`/`afterUpdate`.

```svelte
<script lang="ts">
  let count = $state(0);

  $effect(() => {
    console.log('Count changed:', count);

    return () => {
      console.log('Cleanup before next effect or on destroy');
    };
  });
</script>
```

### $props

Used to declare component props. Replaces `export let`.

```svelte
<script lang="ts">
  let { name, initialCount = 0, children } = $props<{
    name: string;
    initialCount?: number;
    children?: import('svelte').Snippet;
  }>();

  let count = $state(initialCount);
</script>

<h1>Hello {name}!</h1>
{@render children?.()}
```

### $bindable

Used for two-way binding props.

```svelte
<script lang="ts">
  let { value = $bindable() } = $props<{ value: string }>();
</script>

<input bind:value />
```

## Snippets

Snippets allow you to define reusable chunks of UI within a component.

```svelte
{#snippet item(name, price)}
  <li>{name}: ${price}</li>
{/snippet}

<ul>
  {@render item('Apple', 1.5)}
  {@render item('Banana', 0.5)}
</ul>
```

## Advanced Patterns

### State Classes

In Svelte 5, you can use classes to encapsulate reactive logic.

```ts
// counter.svelte.ts
export class Counter {
  count = $state(0);
  double = $derived(this.count * 2);

  increment() {
    this.count += 1;
  }
}
```

```svelte
<script lang="ts">
  import { Counter } from './counter.svelte.js';
  const counter = new Counter();
</script>

<button onclick={() => counter.increment()}>
  {counter.count} / {counter.double}
</button>
```

### Passing Snippets as Props

```svelte
<!-- Parent.svelte -->
<script lang="ts">
  import Child from './Child.svelte';
</script>

<Child>
  {#snippet header()}
    <h1>Custom Header</h1>
  {/snippet}
  
  <p>Main content</p>
</Child>

<!-- Child.svelte -->
<script lang="ts">
  let { header, children } = $props<{
    header: import('svelte').Snippet;
    children: import('svelte').Snippet;
  }>();
</script>

<header>
  {@render header()}
</header>
<main>
  {@render children()}
</main>
```

## Common Svelte 5 Tips

- **Event Handlers**: Use standard HTML attributes like `onclick` instead of `on:click`.
- **Deep Reactivity**: `$state` is deeply reactive for objects and arrays. You don't need to reassign the object to trigger updates.
- **`.svelte.ts` / `.svelte.js`**: Use these extensions for files containing Runes outside of `.svelte` components.
- **`$effect.pre`**: Runs before the DOM is updated (similar to `beforeUpdate`).
- **`$inspect`**: Replaces `$: console.log(...)` for debugging state changes.
- **Untracking**: Use `untrack(() => ...)` inside `$effect` or `$derived` to read state without creating a dependency.
- **Migration**: Most Svelte 4 code still works in Svelte 5 (backward compatibility), but Runes cannot be mixed with legacy reactivity (e.g., `$:`) in the same component.
