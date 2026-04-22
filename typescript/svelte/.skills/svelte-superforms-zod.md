# Svelte Superforms & Zod Skill (Svelte 5)

## Description

Expert skill for managing forms in SvelteKit using `sveltekit-superforms` and `zod` validation. This skill covers schema definition, server-side actions, client-side rehydration, error handling, and Svelte 5 Rune integration. Use this skill when the user needs to create complex forms, validate user input, or handle multi-step form logic.

## Installation

```bash
npm i sveltekit-superforms zod
```

## Schema Definition (Zod)

```ts
// src/lib/schemas.ts
import { z } from 'zod';

export const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
  rememberMe: z.boolean().default(false)
});
```

## Server Action (`+page.server.ts`)

```ts
import { fail } from '@sveltejs/kit';
import { superValidate } from 'sveltekit-superforms';
import { zod } from 'sveltekit-superforms/adapters';
import { loginSchema } from '$lib/schemas';

export const load = async () => {
  const form = await superValidate(zod(loginSchema));
  return { form };
};

export const actions = {
  default: async ({ request }) => {
    const form = await superValidate(request, zod(loginSchema));

    if (!form.valid) {
      return fail(400, { form });
    }

    // Process login...
    console.log(form.data);

    return { form };
  }
};
```

## Component Usage (`+page.svelte`)

```svelte
<script lang="ts">
  import { superForm } from 'sveltekit-superforms';
  import { zodClient } from 'sveltekit-superforms/adapters';
  import { loginSchema } from '$lib/schemas';

  let { data } = $props();

  const { form, errors, enhance, constraints, message } = superForm(data.form, {
    validators: zodClient(loginSchema)
  });
</script>

<form method="POST" use:enhance>
  <div>
    <label for="email">Email</label>
    <input 
      name="email" 
      type="email" 
      bind:value={$form.email} 
      {...$constraints.email} 
    />
    {#if $errors.email}<span class="error">{$errors.email}</span>{/if}
  </div>

  <div>
    <label for="password">Password</label>
    <input 
      name="password" 
      type="password" 
      bind:value={$form.password} 
      {...$constraints.password} 
    />
    {#if $errors.password}<span class="error">{$errors.password}</span>{/if}
  </div>

  <button type="submit">Login</button>
</form>

{#if $message}<p>{$message}</p>{/if}
```

## Superforms Tips

- **`use:enhance`**: Always use `enhance` to prevent full page reloads and enable client-side validation.
- **`$form` Store**: Note that Superforms still uses stores (`$form`, `$errors`) in Svelte 5 for now, but they work perfectly alongside Runes.
- **Nested Data**: Use `z.object().array()` for complex forms and `dataType: 'json'` in `superForm` options.
- **Status Codes**: Use `fail(400, { form })` on the server to return validation errors with the correct HTTP status.
- **Loading State**: Use the `delayed` store from `superForm` to show loading spinners during long-running actions.
- **Zod Adapters**: Always use the correct adapter for your version: `zod(schema)` on server, `zodClient(schema)` on client.
