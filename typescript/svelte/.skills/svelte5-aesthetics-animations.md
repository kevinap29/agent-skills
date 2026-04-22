# Svelte 5 Aesthetics & Animations Skill

## Description

Expert skill for creating stunning, high-performance user interfaces and animations in Svelte 5. Covers `svelte/transition`, `svelte/animate`, GSAP/Motion integration, and advanced Tailwind CSS patterns for a "premium" look. Use this skill when the user wants to polish their UI, add micro-interactions, or create complex page transitions.

## Built-in Transitions

```svelte
<script lang="ts">
  import { fade, fly, slide, scale } from 'svelte/transition';
  import { cubicInOut } from 'svelte/easing';

  let visible = $state(false);
</script>

<button onclick={() => visible = !visible}>Toggle</button>

{#if visible}
  <div 
    transition:fly={{ y: 20, duration: 500, easing: cubicInOut }}
    class="card"
  >
    Premium Content
  </div>
{/if}
```

## List Animations (`animate:flip`)

```svelte
<script lang="ts">
  import { flip } from 'svelte/animate';
  let items = $state([1, 2, 3, 4, 5]);

  function shuffle() {
    items = items.sort(() => Math.random() - 0.5);
  }
</script>

<button onclick={shuffle}>Shuffle</button>

<div class="list">
  {#each items as item (item)}
    <div animate:flip={{ duration: 400 }}>
      Item {item}
    </div>
  {/each}
</div>
```

## Motion Store (Spring & Tweened)

```svelte
<script lang="ts">
  import { spring, tweened } from 'svelte/motion';
  import { cubicOut } from 'svelte/easing';

  const size = spring(1);
  const progress = tweened(0, { duration: 400, easing: cubicOut });
</script>

<button 
  onmouseenter={() => size.set(1.2)} 
  onmouseleave={() => size.set(1)}
  onclick={() => progress.set(1)}
  style="transform: scale({$size})"
>
  Hover & Click Me
</button>

<progress value={$progress}></progress>
```

## Custom CSS Transition Patterns

```svelte
<style>
  .card {
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  }
  .card:hover {
    transform: translateY(-4px) scale(1.02);
    box-shadow: 0 20px 25px -5px rgb(0 0 0 / 0.1);
  }
</style>
```

## Rich Aesthetics Tips

- **Glassmorphism**: Use `backdrop-blur-md bg-white/30 border border-white/20` for modern frosted glass effects.
- **Micro-interactions**: Use `spring` for UI elements that should feel "physical" and responsive.
- **Easing**: Avoid default linear easing. Use `cubicInOut` or custom bezier curves for more natural movement.
- **Staggered Animations**: Use `delay` in transitions for lists to create a "wave" effect when items appear.
- **Dark Mode**: Use the `dark:` utility in Tailwind and ensure colors are tailored for high contrast and readability.
- **Typography**: Use premium fonts (e.g., Inter, Outfit, Montserrat) and careful line-height/letter-spacing for a professional feel.
- **Gradients**: Use mesh gradients or subtle linear gradients (`bg-gradient-to-br from-indigo-500 via-purple-500 to-pink-500`) for visual depth.
