# 2FA Implementation — Frontend (Carebridge)

## Overview

The frontend implements a multi-step login flow with TOTP-based two-factor authentication. The flow is driven by three React components and a dedicated `auth.js` service module that communicates with the backend API.

---

## New Dependency

`qrcode.react` was added to `package.json` to render the TOTP setup QR code as an SVG in the browser.

```json
"qrcode.react": "^4.2.0"
```

---

## Files Changed / Added

### `src/services/auth.js` — Auth service (new file)

Central module for all authentication logic. Exports the following functions:

| Function | Purpose |
|---|---|
| `login({ email, password })` | Step 1 — sends credentials, receives `{ tempToken, requiresTotpSetup }` or `{ tempToken, requires2FA }`. Never writes to localStorage. |
| `setupTotp(tempToken)` | Step 2a — calls `GET /auth/2fa/setup` with the temp token to fetch `{ secret, otpauthUri }` for QR display. |
| `confirmTotp(tempToken, code)` | Step 2b — first-time setup. Posts the scanned 6-digit code to `POST /auth/2fa/confirm`. On success, stores the full session JWT. |
| `verifyTotp(tempToken, code)` | Step 2c — returning users. Posts code to `POST /auth/2fa/verify`. On success, stores the full session JWT. |
| `getToken()` | Reads the JWT from `localStorage`. |
| `getCurrentUser()` | Reads and parses the user object from `localStorage`. |
| `logout()` | Removes `token` and `user` from `localStorage`, fires `auth-changed` event. |
| `notifyAuthChanged()` / `onAuthChanged()` | Custom event bus so `App.jsx` can reactively update auth state without a full page reload. |

**Session storage keys:**
- `localStorage.token` — full 14-day JWT (written only after successful 2FA)
- `localStorage.user` — JSON object `{ id, email, role, name }`

---

### `src/services/api.js` — Axios instance (new file)

Configures a shared Axios instance used by `auth.js` and other service modules.

- `baseURL` is read from `import.meta.env.VITE_API_URL`
- A **request interceptor** automatically attaches `Authorization: Bearer <token>` from `localStorage` to every outgoing request, unless the caller has already set the header manually (which `auth.js` does for temp-token calls during 2FA setup/verify)

---

### `src/pages/Login.jsx` — Multi-step login page (new file)

Replaces the earlier single-step login. Manages a `step` state with three possible values:

```
'credentials'  →  'setup'  (first login, TOTP not yet configured)
'credentials'  →  'verify' (returning user with TOTP already active)
```

**Step: `credentials`**
- Email + password form
- On submit calls `login()` from `auth.js`
- Branches based on the response:
  - `requiresTotpSetup === true` → calls `setupTotp()` to get the QR URI, advances to `setup` step
  - `requires2FA === true` → advances directly to `verify` step

**Step: `setup`**
- Displays a `<QRCodeSVG>` rendered from the `otpauthUri` returned by the backend
- User scans the code with an authenticator app (Google Authenticator, Authy, etc.)
- Accepts a 6-digit numeric input (only digits accepted, enforced via `replace(/\D/g, "")`)
- On submit calls `confirmTotp()`, which stores the full session and navigates to `/`

**Step: `verify`**
- Returning-user TOTP form (no QR code)
- Same 6-digit input
- On submit calls `verifyTotp()`, stores session, navigates to `/`

All steps share `tempToken` state and show inline error messages. The submit button is disabled while a request is in flight (`busy` state) and while the 6-digit code is incomplete.

---

### `src/App.jsx` — App shell (new file)

- Imports `getToken`, `getCurrentUser`, `logout`, `onAuthChanged` from `auth.js`
- Subscribes to the `auth-changed` custom event via `onAuthChanged()` so the navbar and route guards update reactively when login/logout occurs
- Defines a `PrivateRoute` wrapper component that:
  - Redirects to `/login` if no token is present
  - Redirects to `/` if the user's role is not in the `allowedRoles` list
- The `/login` route redirects to `/` if the user is already authenticated

---

## Login Flow Diagram

```
User enters email + password
        │
        ▼
  POST /auth/login
        │
   ┌────┴────────────────────┐
   │                         │
requiresTotpSetup        requires2FA
   │                         │
   ▼                         ▼
GET /auth/2fa/setup     Show verify form
   │
   ▼
Display QR code
   │
User scans + enters code
   │
   ▼
POST /auth/2fa/confirm   POST /auth/2fa/verify
   │                         │
   └──────────┬──────────────┘
              │
       Store JWT + user
       Navigate to "/"
```

---

## Notes

- The old `src/pages/(auth)/Login.jsx` is an earlier single-step login (no 2FA). It is superseded by `src/pages/Login.jsx`, which is the file imported by `App.jsx`.
- The JWT is **only written to `localStorage` after successful 2FA completion** — the temp token used during the setup/verify steps is held in React state and never persisted.
- The Axios interceptor in `api.js` skips attaching the stored token when the caller already provides an `Authorization` header, ensuring temp-token requests during 2FA steps are sent correctly.
