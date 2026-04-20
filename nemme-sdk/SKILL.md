---
name: nemme-sdk
description: >
  Integrate the @nemme/js-sdk for event tracking, page views, forms, and delivery triggers.
  Use when code imports @nemme/js-sdk, or when asked about Nemme tracking, analytics,
  event instrumentation, form delivery, or survey integration. Covers initialization,
  typed event tracking with generated TypeScript types, React hooks, batching, form
  delivery, and troubleshooting.
metadata:
  author: Nemme
  version: 0.8.0
license: MIT
---

# Nemme SDK Integration

Use this skill when implementing event tracking, page view tracking, form delivery, or any integration with the `@nemme/js-sdk`.

## Installation

```bash
npm install @nemme/js-sdk
```

For script tag usage (UMD):

```html
<script src="https://unpkg.com/@nemme/js-sdk@latest/dist/nemme-sdk.umd.js"></script>
<link rel="stylesheet" href="https://unpkg.com/@nemme/js-sdk@latest/dist/js-sdk.css" />
```

## Entry Points

The SDK has four entry points:

| Import | Purpose |
|---|---|
| `@nemme/js-sdk` | Core client — initialization, tracking, flush, destroy |
| `@nemme/js-sdk/react` | React integration — `NemmeProvider`, `useNemme`, `useNemmeContext` |
| `@nemme/js-sdk/forms` | Form types — `Form`, `FormConfig`, `FormError`, question types |
| `@nemme/js-sdk/style.css` | Required stylesheet for form UI rendering |

---

## Core API

### Initialization

```typescript
import NemmeSDK from '@nemme/js-sdk';

const client = NemmeSDK('YOUR_CLIENT_KEY');
await client.init({
  userIdentifier: 'user-123',         // Required: unique user identifier
  secretKey: 'optional-secret',       // Optional: server-side validation
  debug: true,                        // Optional: enable debug logging
  batch: {                            // Optional: batching config (or just `true`/`false`)
    enabled: true,                    // Default: true
    size: 10,                         // Default: 10 events before auto-flush
    delayMs: 10000,                   // Default: 10000ms (10s) between flushes
    sendOnUnload: true,               // Default: true — flush on page unload
  },
  formConfig: {                       // Optional: form display settings
    theme: 'light',                   // 'light' | 'dark'
    zIndex: 9999,
    onFormError: (error) => {         // error.errorType: 'FETCH_ERROR' | 'SUBMISSION_ERROR'
      console.error(error);
    },
  },
  deactivate: false,                  // Optional: disable SDK entirely
  trackUrlParamChanges: true,         // Optional: track query param changes as page views
  getPage: () => window.location.href // Optional: custom page URL getter
});
```

**Important:** `init()` must be called and awaited before `track()`. The client will warn if you track before initialization.

### Chained initialization (alternative)

```typescript
const client = await NemmeSDK.init({
  clientKey: 'YOUR_CLIENT_KEY',
  userIdentifier: 'user-123',
  debug: true,
});
```

### Client Properties

```typescript
client.clientKey         // string — the API key
client.userIdentifier    // string | undefined
client.initialized       // boolean
client.lastInitError     // Error | null — check after init failure
```

---

## Event Tracking

### Track a Custom Event

```typescript
await client.track({
  eventKey: 'button_clicked',
  data: {
    buttonId: 'cta-signup',
    page: 'pricing',
    count: 3,
    premium: true,
  },
});
```

### Track a Page View (manual)

Page views are tracked automatically on navigation (pushState, popstate, visibility change). To track manually:

```typescript
client.trackPageView();
```

### Flush Pending Events

```typescript
await client.flush();
```

### Cleanup

```typescript
client.destroy(); // Removes listeners, restores history methods, flushes pending events
```

---

## Type-Safe Event Tracking with Generated Types

Nemme Studio generates TypeScript type definitions for all events defined in a tracking product. Developers should copy these generated types into their project to get full type safety and IDE autocomplete for event tracking.

### Generated Types Structure

When events are defined in Nemme Studio, the platform generates types in this format:

```typescript
/**
 * Allowed event keys for this tracking product.
 */
type NemmeEventName =
  /**
   * Authentication - User Login
   */
  | 'user_login'
  /**
   * Authentication - User Signup
   */
  | 'user_signup'
  /**
   * E-Commerce - Purchase
   */
  | 'purchase';

type UserLoginProperties = {
  platform: string;
  duration_ms?: number | null;
};

type UserSignupProperties = {
  email: string;
  source?: string | null;
  referral_code?: string | null;
};

type PurchaseProperties = {
  amount: number;
  currency: string;
  item_count?: number | null;
};

type NemmeEventPropertiesMap = {
  user_login: UserLoginProperties;
  user_signup: UserSignupProperties;
  purchase: PurchaseProperties;
};

type NemmeEvent<T extends NemmeEventName = NemmeEventName> = {
  eventKey: T;
  data?: NemmeEventPropertiesMap[T];
};

type NemmeEventProperties<T extends NemmeEventName> = NemmeEventPropertiesMap[T];
```

### How the Types Map to Event Definitions

- Each **event group** (e.g., "Authentication") contains events documented via JSDoc comments above the union member
- Each **event** becomes a member of `NemmeEventName` (using its `eventKey`)
- Each **event property** becomes a field in the corresponding `*Properties` type:
  - Property type `STR` maps to `string`
  - Property type `NUM` maps to `number`
  - Property type `BOOL` maps to `boolean`
  - Required properties are non-optional
  - Optional properties use `?: type | null` syntax
- Property type names use PascalCase derived from the event key (e.g., `user_login` becomes `UserLoginProperties`)

### Using the Generated Types

```typescript
import NemmeSDK from '@nemme/js-sdk';

// Use NemmeEvent<T> with the track method for type-safe tracking
await client.track<NemmeEvent<'purchase'>>({
  eventKey: 'purchase',
  data: {
    amount: 49.99,      // Required number — IDE autocomplete shows this
    currency: 'usd',    // Required string
    item_count: 3,      // Optional number
  },
});

// TypeScript will error if you:
// - Use an unknown eventKey
// - Pass wrong property types
// - Miss required properties
```

### Where to Find the Generated Types

In Nemme Studio, navigate to the tracking product's **Dev** page:
`https://studio.nemme.io/teams/{teamSlug}/products/tracking/{trackingSlug}/dev`

The types are displayed in a code block with copy-to-clipboard functionality. Copy and save them to a file in your project (e.g., `src/types/nemme-events.ts`).

---

## React Integration

### Provider Setup

```tsx
import { NemmeProvider } from '@nemme/js-sdk/react';
import '@nemme/js-sdk/style.css'; // Required for form rendering

function App() {
  return (
    <NemmeProvider
      clientKey="YOUR_CLIENT_KEY"
      config={{
        userIdentifier: 'user-123',
        debug: true,
      }}
    >
      <YourApp />
    </NemmeProvider>
  );
}
```

The provider automatically initializes the client on mount and destroys it on unmount.

### useNemme Hook

```tsx
import { useNemme } from '@nemme/js-sdk/react';

function TrackableButton() {
  const { track, flush, isInitialized, error, client } = useNemme();

  if (error) return <p>SDK error: {error}</p>;
  if (!isInitialized) return <p>Loading...</p>;

  return (
    <button
      onClick={() =>
        track<NemmeEvent<'button_clicked'>>({
          eventKey: 'button_clicked',
          data: { button_id: 'cta-hero' },
        })
      }
    >
      Get Started
    </button>
  );
}
```

### useNemmeContext Hook

For direct access to the context object (lower-level):

```tsx
import { useNemmeContext } from '@nemme/js-sdk/react';

const { client, isInitialized, error } = useNemmeContext();
// Throws if used outside <NemmeProvider>
```

---

## Script Tag / UMD Usage

```html
<script src="https://unpkg.com/react@19/umd/react.production.min.js"></script>
<script src="https://unpkg.com/react-dom@19/umd/react-dom.production.min.js"></script>
<script src="https://unpkg.com/@nemme/js-sdk@latest/dist/nemme-sdk.umd.js"></script>
<link rel="stylesheet" href="https://unpkg.com/@nemme/js-sdk@latest/dist/js-sdk.css" />

<script>
  (async () => {
    const client = await NemmeSDK('YOUR_CLIENT_KEY').init({
      userIdentifier: 'user-123',
      debug: true,
    });

    document.getElementById('cta').addEventListener('click', () => {
      client.track({ eventKey: 'cta_clicked', data: { page: 'home' } });
    });
  })();
</script>
```

React and ReactDOM globals are required for form rendering. If you don't use forms, they can be omitted.

---

## Delivery System (Forms & Surveys)

Deliveries are triggered automatically by the SDK based on rules configured in Nemme Studio. No code is needed beyond initialization — the SDK handles:

1. **Loading deliveries** — fetched on init from `/external/deliveries`
2. **Evaluating triggers** — checked on every page view and custom event
3. **Displaying forms** — rendered as a React overlay with `id="nm"`

### Trigger Types

| Type | Matches when |
|---|---|
| `page_url` | Current URL matches the glob pattern (supports `*`, `**`, `?`) |
| `custom_event` | A tracked event's `eventKey` matches the trigger's `eventKey` |

### Trigger Conditions

Triggers may have additional conditions configured in Nemme Studio that must all be satisfied before a delivery fires. Conditions are evaluated server-side when the SDK calls `/external/deliveries/{id}/should-deliver?triggerId={id}` — no client-side code is required. If any condition fails, the delivery is skipped.

| Condition type | Fields | Semantics |
|---|---|---|
| `days_since` | `daysReference`: `first_seen_at` \| `last_seen_at`<br>`minDays?`: number<br>`maxDays?`: number | Passes when the number of whole days between `now` and the user's first/last tracked event is in `[minDays, maxDays]` (inclusive on both sides; either bound optional). Fails when the user has no tracking events for this product. |

Both bounds accept `0`. A missing bound means unbounded on that side. If neither bound is set, the condition is invalid and rejected by the API.

### URL Pattern Examples

| Pattern | Matches |
|---|---|
| `/pricing` | Exactly `/pricing` |
| `/blog/*` | `/blog/post-1` but not `/blog/2024/post-1` |
| `/docs/**` | `/docs/getting-started`, `/docs/api/reference`, any depth |
| `/products/?` | `/products/a` but not `/products/ab` |

### Form Error Handling

```typescript
import { FormError } from '@nemme/js-sdk/forms';

await client.init({
  userIdentifier: 'user-123',
  formConfig: {
    theme: 'dark',
    zIndex: 10000,
    onFormError: (error: FormError) => {
      // error.errorType is 'FETCH_ERROR' or 'SUBMISSION_ERROR'
      console.error(`Form ${error.errorType}:`, error.message);
    },
  },
});
```

### Form Types (for reference)

```typescript
type Form = {
  name: string;
  slug: string;
  icon?: string;
  startTransition?: StartTransition;
  endTransition?: EndTransition;
  questions: Question[];
};

type QuestionType = 'choice' | 'score' | 'text' | 'ranking';

type TextQuestion = {
  id: number;
  questionType: 'text';
  title: string;
  description?: string;
  image?: string;
  required: boolean;
  order: number;
  limited: boolean;
  maxChars?: number;
};

type ChoiceQuestion = {
  id: number;
  questionType: 'choice';
  title: string;
  description?: string;
  image?: string;
  required: boolean;
  order: number;
  maxSelections: number;
  verticalAlignment: boolean;
  alternatives: { id: number; text: string; order: number }[];
};

type Question = TextQuestion | ChoiceQuestion;
```

---

## Event Data Rules

### Allowed Property Types

| Type | TypeScript | Notes |
|---|---|---|
| String | `string` | Automatically lowercased by the SDK |
| Number | `number` | Integers and floats |
| Boolean | `boolean` | `true` / `false` |

### Data Sanitization (automatic)

- All string values are converted to **lowercase**
- Properties with unsupported types (objects, arrays, functions, null, undefined) are **silently removed** with a warning
- If any value stringifies to `[object Object]`, the **entire event is dropped**

### Validation Behavior

- Events are validated against definitions fetched during init
- Unknown `eventKey` values: **warning logged, event still sent**
- Unknown properties: **warning logged, event still sent**
- Type mismatches: **warning logged, event still sent**
- Validation is permissive — events are sent even with warnings

---

## Batching Behavior

Events are batched by default to reduce network overhead:

- **Single event**: sent via `POST /external/trackings/track`
- **Multiple events**: sent via `POST /external/trackings/batch`
- **Auto-flush triggers**:
  - Batch reaches `size` threshold (default: 10)
  - Timer expires after `delayMs` (default: 10s)
  - Page unload (`beforeunload` / `pagehide`) if `sendOnUnload: true`
- **Manual flush**: `client.flush()` or `flush()` from `useNemme()`
- **Failed flush**: events are prepended back to the queue for retry

To disable batching:

```typescript
await client.init({
  userIdentifier: 'user-123',
  batch: false, // or { enabled: false }
});
```

---

## Troubleshooting

### SDK not tracking events

1. Ensure `init()` is awaited before calling `track()`
2. Check `client.initialized` — if `false`, check `client.lastInitError`
3. Enable `debug: true` to see validation warnings in the console
4. Verify the `clientKey` is correct
5. Verify the `eventKey` matches an event defined in Nemme Studio

### Events silently dropped

- Check for `[object Object]` in event data — this drops the entire event
- Ensure data values are only `string`, `number`, or `boolean`
- Do not pass objects, arrays, or `null` as property values

### Forms not appearing

- Ensure `@nemme/js-sdk/style.css` is imported
- React and ReactDOM must be available (peer dependencies)
- Check that no element with `id="nm"` already exists in the DOM (prevents overlapping)
- Check `formConfig.onFormError` for fetch/submission errors
- Verify delivery triggers match the current page URL or event

### Page views not tracking

- Check `trackUrlParamChanges` if query-param-only changes should be tracked
- The SDK wraps `history.pushState` and `history.replaceState` — if another library also wraps these, ordering may matter
- `visibilitychange` events are used — background tabs won't generate page views until visible

### React hook errors

- `useNemme()` and `useNemmeContext()` must be called inside `<NemmeProvider>`
- Provider auto-initializes on mount — no need to call `init()` manually
- Provider auto-destroys on unmount — no need to call `destroy()`

### String values appear wrong

- All string event data is **automatically lowercased** by the SDK
- If case-sensitive values are needed, this is a known SDK behavior
