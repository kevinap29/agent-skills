# shadcn-svelte with Svelte 5 Skill

## Description

Expert skill for implementing and customizing **shadcn-svelte** components in Svelte 5 projects. This skill covers the integration of Tailwind CSS, Bits UI (headless primitives), and the shadcn-svelte CLI. It focuses on the Svelte 5 versions of components that utilize Runes for state management and props. Use this skill for UI development, accessibility-compliant components, and rapid prototyping.

## Installation

```bash
# Initialize shadcn-svelte in your project
npx shadcn-svelte@next init

# Add a component
npx shadcn-svelte@next add button
npx shadcn-svelte@next add dialog
```

## Component Usage (Svelte 5)

Components in shadcn-svelte for Svelte 5 use the new `$props()` and `$bindable()` runes.

### Button Component

```svelte
<script lang="ts">
  import { Button } from "$lib/components/ui/button";
</script>

<Button variant="outline" size="lg" onclick={() => console.log('Clicked!')}>
  Click Me
</Button>
```

### Dialog (Modal) with State Rune

```svelte
<script lang="ts">
  import * as Dialog from "$lib/components/ui/dialog";
  import { Button } from "$lib/components/ui/button";

  let open = $state(false);
</script>

<Dialog.Root bind:open>
  <Dialog.Trigger asChild let:builder>
    <Button builders={[builder]}>Open Dialog</Button>
  </Dialog.Trigger>
  <Dialog.Content>
    <Dialog.Header>
      <Dialog.Title>Are you sure?</Dialog.Title>
      <Dialog.Description>
        This action cannot be undone.
      </Dialog.Description>
    </Dialog.Header>
    <Dialog.Footer>
      <Button onclick={() => (open = false)}>Cancel</Button>
      <Button variant="destructive">Confirm</Button>
    </Dialog.Footer>
  </Dialog.Content>
</Dialog.Root>
```

### Form Integration (with Formsnap/Zod)

```svelte
<script lang="ts">
  import * as Form from "$lib/components/ui/form";
  import { Input } from "$lib/components/ui/input";
  import { userSchema } from "./schema";
  import { superForm } from "sveltekit-superforms";
  import { zodClient } from "sveltekit-superforms/adapters";

  let { data } = $props();

  const form = superForm(data.form, {
    validators: zodClient(userSchema),
  });

  const { form: formData, enhance } = form;
</script>

<form method="POST" use:enhance>
  <Form.Field {form} name="username">
    <Form.Control let:attrs>
      <Form.Label>Username</Form.Label>
      <Input {...attrs} bind:value={$formData.username} />
    </Form.Control>
    <Form.Description>Your public display name.</Form.Description>
    <Form.FieldErrors />
  </Form.Field>
  <Form.Button>Submit</Form.Button>
</form>
```

## Theming and Customization

### Tailwind Configuration (`tailwind.config.js`)

shadcn-svelte uses CSS variables for theming, allowing for dynamic dark mode switching.

```js
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./src/**/*.{html,js,svelte,ts}'],
  theme: {
    extend: {
      colors: {
        border: "hsl(var(--border) / <alpha-value>)",
        input: "hsl(var(--input) / <alpha-value>)",
        ring: "hsl(var(--ring) / <alpha-value>)",
        background: "hsl(var(--background) / <alpha-value>)",
        foreground: "hsl(var(--foreground) / <alpha-value>)",
        primary: {
          DEFAULT: "hsl(var(--primary) / <alpha-value>)",
          foreground: "hsl(var(--primary-foreground) / <alpha-value>)"
        },
        // ... more variables
      }
    }
  }
}
```

### Using Bits UI Primitives

Many shadcn components are wrappers around Bits UI. You can use Bits UI directly for even more control.

```svelte
<script lang="ts">
  import { Collapsible } from "bits-ui";
</script>

<Collapsible.Root>
  <Collapsible.Trigger>Toggle</Collapsible.Trigger>
  <Collapsible.Content>
    Hidden content revealed!
  </Collapsible.Content>
</Collapsible.Root>
```

## Best Practices

- **Registry**: Always use the `@next` or equivalent version of the CLI for Svelte 5 compatibility.
- **`asChild`**: Use the `asChild` prop on triggers to avoid unnecessary wrapper elements and maintain proper event bubbling.
- **Directives**: Be careful with Svelte 4 style directives (`on:click`) vs Svelte 5 attributes (`onclick`). shadcn-svelte components for Svelte 5 are moving towards standard attributes.
- **Type Safety**: Leverage the generated components in `$lib/components/ui` which are fully typed for TypeScript support.
- **Lucide Icons**: shadcn-svelte pairs perfectly with `lucide-svelte` for iconography.
