# Roles & Access

## The allowlist (`src/auth/roles.js`)
The single source of truth for "who is allowed to sign in, and with what role" — a plain static object, hand-edited per person, no admin UI at this scale:

```js
export const ROLES = {
  'aaron@ninjacart.com': { role: 'ADMIN', projects: ['packtrack'] },
};

export function getRole(email) {
  return ROLES[email.toLowerCase()];
}
```

Role names mirror PackTrack Pro's own model (`ADMIN`, `PM_STORE_EXEC`, `CC_EXEC`, `FC_EXEC`, `CC_DP`, `FC_DP`) rather than inventing a new one.

## What a role actually gates (today)
Which **projects'** tools a token holder can call. `extra.authInfo` carries `role` + `projects`; each project's `index.js` checks its own name is in the caller's `projects` list before running any tool. `ADMIN` is unrestricted across whatever's in `projects`.

## What it deliberately does NOT do yet
Row-level filtering *within* a project's data (e.g. a `CC_EXEC` seeing only their own CC's indents). The SQL tool is intentionally generic/free-form, so enforcing this would need Postgres Row-Level Security tied to a session variable per-request — real, doable, but more than a one-time-activity allowlist justifies right now. Add if/when the hub is used by people below `ADMIN` who shouldn't see everything in a project they're scoped to. Tracked in [[07 - Open Risks]].

## Granting access to a new person
One-time manual edit: add their `@ninjacart.com` email + role + `projects` array to `roles.js`, redeploy. No self-service signup.

## Related Notes
- [[02 - Auth & OAuth Flow]]
- [[07 - Open Risks]]
