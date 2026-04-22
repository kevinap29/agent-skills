# SvelteKit Authentication Skill

## Description

Expert skill for implementing authentication and authorization in SvelteKit using Auth.js (NextAuth) or Lucia Auth. Covers session management, OAuth providers, database adapters, route protection via `hooks.server.ts`, and secure form-based login. Use this skill when the user needs to add a login system, manage user sessions, or restrict access to certain pages.

## Auth.js Setup

```bash
npm install @auth/sveltekit
```

### Server Configuration (`src/auth.ts`)

```ts
import { SvelteKitAuth } from "@auth/sveltekit"
import GitHub from "@auth/sveltekit/providers/github"
import { GITHUB_ID, GITHUB_SECRET } from "$env/static/private"

export const { handle, signIn, signOut } = SvelteKitAuth({
  providers: [GitHub({ clientId: GITHUB_ID, clientSecret: GITHUB_SECRET })],
})
```

### Hooks (`src/hooks.server.ts`)

```ts
export { handle } from "./auth"
```

## Route Protection

```ts
// src/routes/admin/+page.server.ts
import { redirect } from "@sveltejs/kit"

export const load = async (event) => {
  const session = await event.locals.auth()
  if (!session?.user) {
    throw redirect(303, "/auth/signin")
  }
  return { session }
}
```

## Component Usage

```svelte
<script>
  import { signIn, signOut } from "@auth/sveltekit/client"
  let { data } = $props()
</script>

{#if data.session}
  <p>Signed in as {data.session.user?.email}</p>
  <button onclick={() => signOut()}>Sign out</button>
{:else}
  <button onclick={() => signIn("github")}>Sign in with GitHub</button>
{/if}
```

## Manual Auth (Lucia Pattern)

```ts
// src/lib/server/auth.ts
import { Lucia } from "lucia";
import { dev } from "$app/environment";

export const lucia = new Lucia(adapter, {
	sessionCookie: {
		attributes: {
			secure: !dev
		}
	},
    getUserAttributes: (attributes) => {
		return {
			username: attributes.username
		};
	}
});

declare module "lucia" {
	interface Register {
		Lucia: typeof lucia;
		DatabaseUserAttributes: {
			username: string;
		};
	}
}
```

## Authentication Tips

- **Hooks**: Use `hooks.server.ts` to perform high-level checks (e.g., checking if a user is authenticated) before a request even reaches a route.
- **CSRF**: SvelteKit has built-in CSRF protection for form actions. Ensure it is enabled.
- **Secure Cookies**: Always use `HttpOnly` and `Secure` flags for session cookies.
- **Database Adapters**: Use official adapters (Prisma, Drizzle, etc.) to store user data and sessions.
- **Environmental Variables**: Store client IDs and secrets in `.env` and access them via `$env/static/private`.
- **RBAC**: Implement Role-Based Access Control by checking user roles in `locals` or database during the `load` function.
