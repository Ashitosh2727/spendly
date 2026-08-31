# Spec: Registration

## Overview
Spendly currently renders a static `register.html` form at `GET /register`, but submitting it does nothing — there is no backend logic to validate input, hash the password, or persist the new user. This step wires up account creation: handling the form POST, validating and hashing credentials, inserting the user via `database/db.py`, and surfacing duplicate-email errors back to the form. It builds directly on the schema and `get_db()` helper established in Step 1, and unblocks Step 3+ (logout, profile, login) which all require real user rows to operate against.

## Depends on
- Step 1 — Database setup (`users` table, `get_db()`, `init_db()`) — complete.

## Routes
- `GET /register` — render the registration form — public (already implemented, no change in behavior)
- `POST /register` — validate input, create the user, redirect to login on success or re-render the form with an error — public

## Database changes
No schema changes. `users` table already has the required columns (`name`, `email`, `password_hash`, `created_at`) per `database/db.py`.

New functions to add to `database/db.py` (logic must not live in `app.py`):
- `get_user_by_email(email)` — `SELECT * FROM users WHERE email = ?`, returns a row or `None`
- `create_user(name, email, password_hash)` — parameterized `INSERT INTO users (name, email, password_hash) VALUES (?, ?, ?)`, returns the new user's id

## Templates
- **Create:** none
- **Modify:** `templates/register.html` — change the hardcoded `action="/register"` to `action="{{ url_for('register') }}"` (CLAUDE.md forbids hardcoded URLs); repopulate `name`/`email` field values from the submitted form on validation errors so the user doesn't retype everything

## Files to change
- `app.py` — change `register()` to accept `methods=["GET", "POST"]`; on POST, validate fields, check for an existing email via `get_user_by_email`, hash the password with `werkzeug.security.generate_password_hash`, call `create_user`, then redirect to `url_for('login')`
- `database/db.py` — add `get_user_by_email()` and `create_user()`
- `templates/register.html` — fix hardcoded form action, repopulate values on error

## Files to create
None

## New dependencies
No new dependencies

## Rules for implementation
- No SQLAlchemy or ORMs
- Parameterised queries only
- Passwords hashed with werkzeug (`generate_password_hash`), never stored or logged in plaintext
- Use CSS variables — never hardcode hex values
- All templates extend `base.html`
- Server-side validation required even though the form has `required`/`type=email` attributes (HTML validation is not a security boundary): reject empty name, invalid email format, and password under 8 characters
- Duplicate email must re-render `register.html` with the existing `auth-error` block — never a raw string or a 500

## Definition of done
- [ ] `GET /register` still renders the form unchanged
- [ ] Submitting valid name/email/password creates a row in `users` with a hashed (not plaintext) password
- [ ] Successful registration redirects to `/login`
- [ ] Submitting an already-registered email re-renders `register.html` with an error and does not insert a duplicate row
- [ ] Submitting a password under 8 characters is rejected with an error, no row inserted
- [ ] Submitting an empty name or malformed email is rejected with an error, no row inserted
- [ ] Form action in `register.html` uses `url_for('register')`, not a hardcoded path
- [ ] App starts and runs on port 5001 without errors after the change
