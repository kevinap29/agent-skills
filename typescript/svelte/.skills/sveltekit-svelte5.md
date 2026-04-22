# SvelteKit with Svelte 5 Skill

## Description

Expert skill for SvelteKit development leveraging Svelte 5 Runes. This skill covers file-based routing, server-side rendering (SSR), data loading, form actions, API routes, and integration with Svelte 5's reactive system. Use this skill for any SvelteKit-specific questions including routing structure, `+page` vs `+layout`, server-side logic, and SEO optimization in Svelte 5 projects.

## Project Structure

```text
src/
├── lib/             # Shared code (components, utilities)
├── routes/          # File-based routing
│   ├── +layout.svelte
│   ├── +page.svelte
│   ├── api/
│   │   └── data/+server.ts
│   └── blog/
│       ├── [slug]/
│       │   ├── +page.svelte
│       │   └── +page.server.ts
├── app.d.ts         # Type definitions
└── app.html         # Page template
```

## Data Loading

In Svelte 5, the `data` prop is received via `$props()`.

### Server Load (`+page.server.ts`)

```ts
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ params, locals }) => {
    return {
        post: {
            title: 'Svelte 5 is here',
            content: '...'
        }
    };
};
```

### Component Usage (`+page.svelte`)

```svelte
<script lang="ts">
    import type { PageData } from './$types';
    
    let { data } = $props<{ data: PageData }>();
    
    // Create reactive state from data if needed
    let post = $derived(data.post);
</script>

<h1>{post.title}</h1>
<div>{@html post.content}</div>
```

## Form Actions

### Server Action (`+page.server.ts`)

```ts
import { fail, type Actions } from '@sveltejs/kit';

export const actions: Actions = {
    default: async ({ request }) => {
        const data = await request.formData();
        const email = data.get('email');

        if (!email) {
            return fail(400, { email, missing: true });
        }

        // Process data...
        return { success: true };
    }
};
```

### Component Usage (`+page.svelte`)

```svelte
<script lang="ts">
    import { enhance } from '$app/forms';
    import type { ActionData } from './$types';

    let { form } = $props<{ form: ActionData }>();
</script>

<form method="POST" use:enhance>
    <input name="email" type="email" />
    {#if form?.missing}
        <p class="error">Email is required</p>
    {/if}
    <button>Subscribe</button>
</form>

{#if form?.success}
    <p>Success!</p>
{/if}
```

## API Routes (`+server.ts`)

```ts
import { json } from '@sveltejs/kit';
import type { RequestHandler } from './$types';

export const GET: RequestHandler = async () => {
    return json({ message: 'Hello from Svelte 5 API' });
};

export const POST: RequestHandler = async ({ request }) => {
    const body = await request.json();
    return json({ received: body }, { status: 201 });
};
```

## State Management in SvelteKit

### Using `$state` in `$lib`

```ts
// src/lib/settings.svelte.ts
export class Settings {
    theme = $state('dark');
    
    toggle() {
        this.theme = this.theme === 'dark' ? 'light' : 'dark';
    }
}

export const settings = new Settings();
```

### Page Navigation

```svelte
<script lang="ts">
    import { goto, invalidate, invalidateAll } from '$app/navigation';
</script>

<button onclick={() => goto('/about')}>About</button>
<button onclick={() => invalidate('/api/data')}>Refresh Data</button>
```

## SvelteKit + Svelte 5 Tips

- **Data Prop**: Always use `$props()` to receive `data` and `form` in `+page.svelte`.
- **SSR Rehydration**: Remember that `$effect` only runs on the client. Use `onMount` or `$effect` for client-side only logic.
- **`$page` Store**: The `$page` store is still useful for accessing URL params and route info, though you can often get these from `data`.
- **Runes in Load**: Do **not** use Runes (`$state`, etc.) inside `load` functions or server-side code. They are for UI reactivity.
- **`enhance`**: Use the `use:enhance` action for seamless form submissions without page reloads.
- **Layouts**: Use `+layout.svelte` for persistent UI (navigation, footers) and state that should survive page changes.
