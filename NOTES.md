# NOTES.md — Sanctum Sanctorum Bookstore

## Deployed URL

> **Live URL:** https://sanctum-sanctorum-mjgc.onrender.com
>
> Seeded members to test with:
> - Wong Li (supreme) — id 1
> - Christine Palmer (master) — id 2
> - Jonathan Pangborn (adept) — id 3
> - Sara Lin (apprentice) — id 4

---

## What I Finished

All 202 tests pass. Every feature from the spec is implemented:

### Books
- ISBN-13 checksum validation (weights 1,3,1,3…; check digit verification)
- Duplicate ISBN detection → 409
- `GET /books` search: fixed to match title **OR** author (was title-only)
- `min_price` / `max_price` inclusive filters
- Sorting by `title`, `-title`, `price`, `-price` with id tiebreaker
- `total` count fixed to count all matches before pagination (was counting only the current page)
- `PATCH /books/{id}` endpoint — partial updates, isbn silently ignored

### Members
- Email normalization: strip whitespace + lowercase before validation and storage
- Duplicate email detection (case-insensitive) → 409
- Fixed `tier_at_least()`: changed `>` to `>=` so a `master`-tier member correctly passes the "at least master" check
- `GET /members/{id}/stats` — aggregates paid orders, active/overdue loans, and late fees

### Orders
- Schema validation: reject empty items list and duplicate book_ids → 422
- `calculate_discount_percent()`: tier-based discount + 5% bulk bonus when total quantity ≥ 10
- `create_order()`: full validation chain (member lookup → book lookup → restriction check → all-or-nothing stock check → stock decrement → price snapshot → discount calculation)
- Fixed `cancel_order()`: now restores stock for every item (was only changing status)

### Loans
- Completed Loan model: added `due_at`, `returned_at` (nullable), `late_fee_cents` columns
- `loan_status()`: computed at read time (returned → overdue → active)
- `create_loan()`: 6-step validation (member/book 404, restriction 403, overdue check, duplicate loan check, tier limit check, stock check)
- `return_loan()`: sets returned_at, calculates late fee (25¢/day ceil, capped at book price), restores stock
- `list_member_loans()`: with optional computed status filter
- Boundary rule: at exactly `due_at`, loan is still active (strict `>` comparison)

### Reports
- `GET /reports/top-books`: aggregates quantities from paid orders, sorted by copies_sold desc then title asc

---

## What I Did Not Finish / Skipped

- **Optional extras** from ASSIGNMENT.md (concurrent order handling, `GET /members` with pagination, additional edge-case tests) were not implemented — I focused on getting all required features correct first.

---

## Architectural Decisions and Trade-offs

### Layered architecture (Router → Service → Model)
I preserved the existing layered pattern throughout. Routers stay thin (parse request, delegate, return). All business logic lives in services. This keeps the code testable and makes it clear where each rule is enforced.

### Validation order matches the spec exactly
The spec prescribes a specific order of checks for orders and loans (e.g., 404 before 403 before 409). I followed this precisely rather than combining checks, even though some could be done in parallel. This ensures deterministic, spec-compliant error responses.

### All-or-nothing stock checks
For `create_order`, I verify stock for **all** items before decrementing **any** of them. This prevents partial stock changes if one book is out of stock. The same transactional approach applies to cancellation (all stock is restored).

### Computed loan status
Loan status (`active`, `overdue`, `returned`) is computed at read time based on `returned_at` and the current clock, rather than stored. This avoids stale data — a loan that becomes overdue overnight doesn't need a background job to update its status.

### Late fee calculation
I used `math.ceil()` on the total seconds divided by 86400 (seconds/day) to implement "any partial day counts as a full day". This handles edge cases correctly (e.g., 14 days and 1 second late = 1 day, 14 days and 23 hours late = 1 day, exactly 15 days late = 1 day).

### Database compatibility
I modified `db.py` to conditionally apply `check_same_thread` only for SQLite URLs. This allows the same codebase to work with SQLite locally (for tests) and Postgres in production, without environment-specific code branches.

### Reuse of service functions
Services reuse each other where appropriate (e.g., `create_order` calls `get_member` and `ensure_can_access_restricted` from the members service). This avoids duplicating validation logic.

---

## Deployment

**Platform choice: Render (backend) + Neon/Supabase (Postgres)**

I chose Render because it natively supports Python web services without requiring a custom Dockerfile or ASGI adapter (unlike Vercel, which needs extra plumbing for FastAPI). The free tier is sufficient for this project.

For the database, I moved from SQLite to Postgres because serverless platforms have no persistent local disk — SQLite files would silently vanish between deploys/restarts.

The `SANCTUM_DATABASE_URL` environment variable is set in the Render dashboard to point to the hosted Postgres instance. Locally, it defaults to SQLite so `uv run pytest` continues to work unchanged.

**Steps taken:**
- Made `db.py` Postgres-compatible (conditional `connect_args`)
- Added `render.yaml` for Render's blueprint deployment
- Added `requirements.txt` with `psycopg2-binary` for Postgres connectivity

---

## AI Usage

### Tools used
- **Google Antigravity (Gemini)** — used as a pair-programming assistant throughout the project

### What I used it for
- **Understanding the codebase**: Initial exploration of the project structure, identifying all TODOs, bugs, and missing features
- **Implementation**: Writing service functions following the spec's validation rules and business logic
- **Debugging**: Running tests and interpreting failures to iterate on fixes
- **Deployment prep**: Setting up Postgres compatibility and deployment configuration

### Where AI was wrong or unhelpful
- The AI initially set up the email validator to only add `strip().lower()` but didn't catch that the original code already had the validator structure — it just needed the normalization line added. A minor issue but it shows the AI sometimes over-explains simple fixes.
- When writing the books service, the AI initially used a full file rewrite instead of targeted edits. While the result was correct, targeted edits would have been cleaner for the git diff and more representative of real development.

### My approach
I used AI as a coding assistant but reviewed every change against the spec before accepting it. I verified each phase with the test suite and ensured I understood the reasoning behind each implementation choice (validation ordering, all-or-nothing semantics, computed vs stored status, etc.).
