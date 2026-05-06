# CLAUDE.md

> Persistent context for Claude Code. Read this at the start of every session.
> Last updated: project kickoff.

---

## Project Overview

**ManPhone** is a React Native mobile app (iOS + Android) that helps users respond
to messages and calls from their most important contacts — and only those contacts.
Everything else is invisible to the app.

**Tagline:** Stop missing what matters.

**Core idea:** User defines a VIP circle of 3–5 people. Only they can trigger alerts.
Alerts escalate if ignored, then become Red Alerts that cannot be swiped away.
One-tap pre-set replies take 2 seconds. The app is silent 95% of the time.

**Solo founder project.** I work part-time, evenings and weekends. Optimize for
shipping speed and minimum operational complexity, not theoretical perfection.

---

## Tech Stack (locked in — do not propose alternatives)

| Layer | Tool |
|---|---|
| Mobile framework | React Native via Expo (managed workflow) |
| Language | TypeScript (strict mode) |
| Navigation | Expo Router (file-based) |
| State | Zustand |
| Styling | NativeWind (Tailwind for RN) |
| Database + Auth | Supabase |
| Backend API | Node.js + Express on Railway |
| Push notifications | Expo Notifications |
| SMS / OTP | Twilio (Verify for OTP, Programmable SMS for replies) |
| Subscriptions | RevenueCat |
| Error tracking | Sentry |
| Marketing site | Next.js on Vercel |

If a session asks you to introduce a new dependency outside this list, **pause
and ask me first.**

---

## Always

- Use TypeScript strict mode. No `any`. No `// @ts-ignore` without a comment explaining why.
- Functional components and hooks only. No class components.
- Use named exports for components, default export only for pages (Expo Router requirement).
- Extract any inline style longer than 5 lines to a StyleSheet or NativeWind classes.
- Reference `TECH_SPEC.md` (in repo root) for database schema, API endpoints, and alert escalation rules.
- Reference `WIREFRAMES/` folder (or screenshots) when building UI for visual fidelity.
- Use the dark theme palette consistently (see Design Tokens below).
- Add loading states, empty states, and error states to every screen with async data.
- Wrap async operations in try/catch and surface errors via toast or inline message.
- Add Sentry breadcrumbs for important user actions (alert fired, reply sent, VIP added).
- Write small, composable functions. If a function is over 40 lines, extract helpers.
- Keep components under 200 lines. Split into subcomponents if longer.
- Use early returns to reduce nesting.

## Never

- Never use `localStorage`, `sessionStorage`, or any web storage API. We are in React Native.
- Never use HTML elements like `<div>`, `<button>`, `<input>`. Use React Native components.
- Never commit `.env`, `.env.local`, `secrets.json`, or any file containing keys.
- Never log user phone numbers, message content, or auth tokens to console or Sentry.
- Never add features that are not in the current session's scope. Ask first.
- Never reorganize folders or rename files unless I explicitly ask. Stability matters.
- Never install a new major dependency without flagging it for me first.
- Never push directly to `main`. Always use a feature branch.
- Never modify the database schema without creating a versioned migration file.
- Never implement custom encryption. Trust Supabase + Twilio for sensitive data flows.
- Never bypass Row Level Security in Supabase queries from the client.

---

## Design Tokens

Use these everywhere. If you need a new color or font, ask me before adding it.

```ts
// constants/theme.ts
export const colors = {
  bg:        '#060606',  // app background
  surface:   '#111111',  // cards, modals
  surface2:  '#161616',  // nested cards
  border:    '#1e1e1e',  // dividers, card borders
  
  text:      '#f0ece4',  // primary text
  textDim:   '#888888',  // secondary text  
  textFaint: '#444444',  // tertiary, captions
  textGhost: '#222222',  // disabled, placeholders
  
  accent:    '#e8ff3c',  // primary actions, brand
  
  red:       '#ff6b6b',  // Red Alert level + errors
  orange:    '#ff9f43',  // Urgent alert level + warnings
  teal:      '#4ecdc4',  // Standard alert level + info
  green:     '#a8d060',  // success states
  purple:    '#c77dff',  // snooze states
};

export const fonts = {
  display: 'BebasNeue',         // big numbers, hero text
  heading: 'DMSerifDisplay',    // screen titles
  body:    'DMSans',            // everything else
  bodyMed: 'DMSans-Medium',     // emphasized body
  bodyBold: 'DMSans-SemiBold',  // strong emphasis
};

export const radius = {
  sm: 6,
  md: 10,
  lg: 14,
  pill: 999,
};

export const spacing = {
  xs: 4,
  sm: 8,
  md: 12,
  lg: 16,
  xl: 24,
  xxl: 32,
};
```

---

## Folder Structure

```
/app                       → Expo Router pages
  /(auth)                  → unauthenticated routes (phone, verify)
  /(tabs)                  → main app tabs (home, vips, history, settings)
  /alert/[id]              → priority alert screen
  /alert/[id]/reply        → quick reply screen
  /paywall                 → subscription paywall
  /legal                   → privacy + terms

/components                → reusable components (PascalCase.tsx)
  /ui                      → primitives: Button, Card, Avatar, Badge
  /vip                     → VIP-specific: VIPCard, VIPForm, AlertLevelPicker
  /alert                   → Alert-specific: AlertHeader, ReplyChip

/lib                       → utilities and clients
  supabase.ts              → Supabase client init
  /api                     → typed API client functions
    vips.ts
    replies.ts
    alerts.ts
    auth.ts

/stores                    → Zustand stores
  authStore.ts
  vipStore.ts
  alertStore.ts

/types                     → shared TypeScript types
  database.ts              → mirrors Supabase schema
  api.ts                   → request/response shapes

/constants                 → theme.ts, config.ts, copy.ts (UI strings)

/hooks                     → custom hooks (use-prefix)

/server                    → Node.js backend (separate, deploys to Railway)
  /src
    index.ts               → Express app
    engine.ts              → AlertEngine class
    /routes
    /lib
  /migrations              → SQL migration files for Supabase

/supabase                  → Supabase config + migrations
  /migrations
```

---

## Database Conventions

- Every table has `id uuid PRIMARY KEY DEFAULT gen_random_uuid()`.
- Every table has `created_at timestamptz DEFAULT now()`.
- Mutable tables also have `updated_at timestamptz DEFAULT now()` with a trigger.
- All foreign keys use `ON DELETE CASCADE` unless data should be preserved.
- Every table has Row Level Security (RLS) enabled with policies for SELECT/INSERT/UPDATE/DELETE.
- Policy pattern: `auth.uid() = owner_id` (users only access their own data).
- Use `text` for strings, never `varchar`. Use `CHECK` constraints for enums.
- Migration filenames: `YYYYMMDDHHMMSS_short_description.sql` (Supabase CLI format).

---

## API Conventions (Node.js backend on Railway)

- Base URL pattern: `/v1/{resource}`.
- Auth: every authenticated route requires `Authorization: Bearer <supabase_jwt>`.
- Validate JWT server-side using Supabase's `auth.getUser(token)`.
- Use Zod for request body validation.
- Return JSON. Errors follow shape: `{ error: { code: string, message: string } }`.
- Status codes: 200 success, 201 created, 400 bad input, 401 unauth, 403 forbidden, 404 not found, 500 server error.
- Never expose Supabase service role key to the client. Only the backend has it.
- Rate limit by user ID, not IP — phone networks share IPs.

---

## Security Rules

- Phone numbers stored in E.164 format only (`+15551234567`).
- Twilio webhook signatures must be verified on every inbound request.
- Reply messages are stored in `reply_log` for 90 days then auto-deleted via cron.
- Alert events are purged after 30 days via cron.
- User deletion (`DELETE /user`) cascades through every table — verify with a test.
- No analytics SDKs in MVP. Sentry only, with PII scrubbing turned on.

---

## Performance Rules

- No animation longer than 300ms unless it's the Red Alert pulse.
- Use `FlatList` for any list over 10 items, never `map()` inside `ScrollView`.
- Use `useMemo`/`useCallback` only when measurable. Don't add prematurely.
- Images: always specify width/height. Use `expo-image` for caching.
- Avoid re-rendering lists when only one item changes — use stable keys + memo.

---

## What "Done" Means

A session is only complete when:

1. The feature works end-to-end on the iOS simulator.
2. There are no TypeScript errors (`npx tsc --noEmit` passes).
3. There are no ESLint errors (`npm run lint` passes).
4. Empty / loading / error states are handled.
5. The session's `git commit` message clearly describes what was added.
6. Any new env vars are documented in `.env.example`.
7. The README is updated if setup steps changed.

---

## Working Style I Prefer

- **Pause and ask** when scope expands. I'd rather ship 80% of session 3 cleanly than 100% of sessions 3 and 4 messy.
- **Show your plan** before writing files when a task has > 3 steps.
- **Group related changes** into one commit, not one commit per file.
- **Surface trade-offs** instead of silently picking. "I can do X (faster) or Y (cleaner) — which do you prefer?"
- **Suggest tests** for tricky logic (the alert engine, especially), but don't over-test simple components.
- **Be honest** about what didn't work. If something is hacky, tell me — I'd rather know.

---

## Things Specific to ManPhone

### The alert engine is the heart of the app
If a session is touching alert escalation, double-check the spec. The rules around
attempt_count, snooze, status transitions, and timer cancellation are easy to break.
There are integration tests planned in `/server/__tests__/engine.test.ts` — keep them green.

### iOS critical alerts are deferred
Apple requires a special entitlement that takes weeks to approve. For MVP, use
time-sensitive notifications (`interruptionLevel: 'timeSensitive'`) instead.
Mark TODOs where critical alerts will eventually go.

### The "Gifted" SKU is the killer feature
The growth model relies on partners gifting ManPhone to the men in their life.
The gift code flow is mission-critical — handle it with care, especially the
redemption path. A broken gift code = a refund + a churned customer.

### Don't over-engineer onboarding
3 steps maximum: phone → OTP → add first VIP. The user is suspicious of yet
another app. Get them to a working state in under 60 seconds.

---

## Open Questions / Decisions Pending

These are not blocking but should be resolved before launch. If a session bumps into one, ask.

- [ ] Final pricing: Plus at $3.99 vs $4.99? A/B test post-launch.
- [ ] Should free users see a "history" tab at all, or paywall it?
- [ ] How do we handle a VIP who doesn't have a smartphone (SMS-only)?
- [ ] Localization strategy — English only at launch? French Canada is a likely 2nd market.
- [ ] Apple Sign In — do we need to offer it alongside phone auth? (Apple guideline 4.8.)

---

*End of CLAUDE.md. Treat the rules in "Always" and "Never" as binding.*
