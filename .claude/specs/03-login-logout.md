# Spec: Login and Logout

## Overview
`GET /login` already renders a working sign-in form and `GET /logout` is currently a stub that returns a raw string. This step wires up real authentication: verifying credentials against the `users` table created in Step 1, establishing a signed Flask session on success, and clearing that session on logout. It also updates the shared nav so a logged-in user can actually reach `/logout` and sees their signed-in state, since nothing currently links to it. This is the step that turns the account created in Step 2 (Registration) into something a user can actually sign into and out of, and unblocks Step 4 (Profile) and beyond, which all require a real logged-in session to identify the current user.

## Depends on
- Step 1 — Database setup (`users` table, `get_db()`) — complete.
- Step 2 — Registration (`get_user_by_email()`, real user rows with hashed passwords) — complete.

## Routes
- `GET /login` — render the sign-in form — public
- `POST /login` — validate credentials, start a session, redirect — public
- `GET /logout` — clear the session, redirect to the landing page — logged-in (safe to hit while logged out too; it's a no-op clear + redirect, not an error)

**Amended after implementation:** an already-logged-in user could still reach `GET /login` and `GET /register` and see the auth forms again. Added a guard at the top of both routes (`if session.get("user_id"): return redirect(url_for("landing"))`) so a signed-in user visiting either is bounced to `/` instead.

## Database changes
No schema changes. Login only needs to read the existing `users.password_hash` column via the existing `get_user_by_email()` (Step 2, `database/db.py`) — reused as-is, no new query needed. No new `database/db.py` functions required for this step.

## Templates
- **Create:** none
- **Modify:**
  - `templates/login.html` — change hardcoded `action="/login"` to `action="{{ url_for('login') }}"` (CLAUDE.md forbids hardcoded URLs, same pre-existing debt pattern fixed in Step 2's `register.html`); repopulate the `email` field (not `password`) on a failed login attempt
  - `templates/base.html` — nav currently always shows "Sign in" / "Get started" regardless of session state, and nothing links to `/logout`. Wrap the nav links in `{% if session.user_id %}...{% else %}...{% endif %}`: logged-in shows a "Log out" link (`url_for('logout')`); logged-out keeps the existing "Sign in" / "Get started" links unchanged.

## Files to change
- `app.py` —
  - Set `app.secret_key` (required for Flask's session cookie signing; currently unset, so `session[...]` would fail)
  - Change `login()` to accept `methods=["GET", "POST"]`; on POST, look up the user via `get_user_by_email`, verify the password with `werkzeug.security.check_password_hash`, set `session["user_id"] = user["id"]` on success and redirect to `/` (landing), or re-render `login.html` with a generic error on failure
  - **Amended after implementation:** originally redirected to `/profile`, but that route is still Step 4's unstyled stub (bare string, no nav) — landing on it right after login left the user unable to see they were signed in or find the logout link. Changed to redirect to `/` instead, where the nav visibly reflects the logged-in state.
  - Change `logout()` to clear the session (`session.clear()`) and redirect to the landing page (`/`), replacing the current `"Logout — coming in Step 3"` stub string
- `templates/login.html` — fix hardcoded form action, repopulate email on error
- `templates/base.html` — conditional nav based on session state

## Files to create
None

## New dependencies
No new dependencies — Flask's session support (`itsdangerous` signed cookies) ships with Flask itself, already in `requirements.txt`.

## Rules for implementation
- No SQLAlchemy or ORMs
- Parameterised queries only
- Passwords verified with werkzeug's `check_password_hash` against the stored hash — never compare plaintext, never re-hash and compare hashes
- Use CSS variables — never hardcode hex values
- All templates extend `base.html`
- Use one generic error message ("Invalid email or password.") for both "no such email" and "wrong password" cases — do not reveal which one was wrong, to avoid leaking which emails are registered (this intentionally differs from Step 2's registration flow, where naming the specific "email already exists" error is normal, expected UX)
- Do not implement or modify the `/profile`, `/expenses/*` stub route bodies — this step only needs `/profile` as a redirect *target*, not an implemented page

## Definition of done
- [ ] `GET /login` still renders the form unchanged
- [ ] Submitting the seeded demo credentials (`demo@spendly.com` / `demo123`) logs in successfully and redirects to `/`
- [ ] Submitting a correct email with a wrong password shows the generic error, no session set
- [ ] Submitting an email that isn't registered shows the same generic error (not a different message)
- [ ] After a successful login, the nav (visible on any page) shows "Log out" instead of "Sign in" / "Get started"
- [ ] Visiting `/logout` clears the session and redirects to `/`; nav reverts to "Sign in" / "Get started"
- [ ] Visiting `/logout` while already logged out doesn't error — just redirects to `/`
- [ ] A logged-in user visiting `/login` or `/register` is redirected to `/`, not shown the form again
- [ ] App starts and runs on port 5001 without errors after the change
