# Member 2 — IBM Bob 2.0 Refactoring Workflow

> **Contributor:** Partho Das (Member 2)  
> **Repository:** `subhradeepkundu270305/ibm-bob-2.0-hackathon`  
> **Branch refactored:** `feature/code-smells-refactor` → merged into `main`  
> **Tooling:** IBM Bob 2.0 (Agent mode)

---

## Overview

This document records the end-to-end refactoring contribution made by Member 2
to the URL Shortener project. Starting from a 641-line monolithic `index.js`,
the codebase was progressively improved across six IBM Bob 2.0 prompting sessions,
each targeting a specific category of technical debt. The final result is a
modular, tested, injection-safe application whose architecture is documented
below.

The companion audit document [`docs/code-smells.md`](code-smells.md) contains
the original static-analysis catalogue. This document focuses on the
**workflow**, **decisions made**, and **resulting structure**.

---

## IBM Bob 2.0 Prompting Sessions

### Prompt 2.1 — Code Smell Audit

**Task given to Bob:**  
Review `index.js` and identify the five most critical issues blocking production
readiness. Produce a structured document with exact line references, risk
assessments, and concrete refactoring recipes.

**Bob produced:** [`docs/code-smells.md`](code-smells.md) — a prioritised
catalogue of five smells: unparameterised SQL, duplicated validation, deeply
nested conditionals, silently swallowed errors, and scattered magic values.

**Decision:** Address smells in dependency order — validation first (DRY),
then nesting (depends on validation extraction), then constants (prepares all
subsequent diffs), then SQL injection and error handling (mechanical but
safety-critical), then modularisation.

---

### Prompt 2.2 — Extract `validateUrl()` (DRY Violation)

**Task given to Bob:**  
Extract the copy-pasted 8-step URL validation waterfall repeated in
`POST /api/shorten` and `PUT /api/urls/:code` into a single reusable
`validateUrl(url)` helper. Update both routes to use it.

**Bob produced:**
- New `validateUrl(url)` function returning `null` (valid) or an error string.
- Both routes reduced to a single three-line validation block.
- The character-by-character space-scan loop replaced with `url.includes(' ')`.

**Before / after (POST handler, validation section):**
```js
// BEFORE — 40-line pyramid, duplicated in PUT
if (typeof temp === 'string') {
  if (temp.length > 0) {
    if (temp.length <= 2048) {
      // … 5 more levels …

// AFTER — one call
var validationError = validateUrl(targetUrl);
if (validationError) { return res.status(400).send({ err: validationError }); }
```

---

### Prompt 2.3 — Guard Clauses (Arrow-of-Doom Nesting)

**Task given to Bob:**  
Replace all remaining deeply nested `if/else` pyramids across all route
handlers with guard clauses and early returns. Maintain identical validation
criteria and status codes.

**Bob produced:** All six route handlers flattened. Maximum nesting depth
reduced from 9 levels to 2 (the DB callback level). Key patterns applied:

| Pattern | Example |
|---|---|
| Invert and early-return | `if (!shortCode) { return res.status(400)… }` |
| Merge sibling conditions | `if (len < 4 \|\| len > 10) { … }` |
| Guard inside callback | `if (!urlRecord) { return res.status(404)… }` |
| `continue` over nested `if` | `if (!row) { continue; }` in list loop |

---

### Prompt 2.4 — Named Constants and Semantic Renaming

**Task given to Bob:**  
Extract all magic numbers and strings into named constants. Rename all
ambiguous variables (`x`, `res2`, `temp`, `tempCode`, `arr`, `thing`, `q`,
`obj`, etc.) to standard, intention-revealing names.

**Bob produced:**

**Constants introduced:**
| Constant | Value | Replaces |
|---|---|---|
| `DB_PATH` | `'database.sqlite'` / `process.env.DB_PATH` | inline string |
| `PORT` | `process.env.PORT \|\| 5000` | hardcoded `5000` |
| `BASE_URL` | `process.env.BASE_URL \|\| 'http://localhost:' + PORT` | 4 inline strings |
| `MAX_URL_LENGTH` | `2048` | inline `2048` |
| `CODE_MIN_LENGTH` | `4` | inline `4` × 3 routes |
| `CODE_MAX_LENGTH` | `10` | inline `10` × 3 routes |
| `CODE_GEN_LENGTH` | `6` | inline `6` |
| `BASE62_CHARSET` | `'abcdef…0123456789'` | inline `str` variable |

**Variable renames (selected):**
| Old | New |
|---|---|
| `x` / `res2` | `req` / `res` |
| `temp` (URL value) | `targetUrl` |
| `temp` (code param) | `shortCode` |
| `tempCode` | `shortCode` |
| `arr` / `arr2` / `thing` | `rows` / `records` / `row` |
| `q` / `q2` | `sql` / `updateSql` |
| `urlErr` | `validationError` |

---

### Prompt 2.5 — Parameterised SQL and Error Handling

**Task given to Bob:**  
Replace all raw string-concatenation SQL queries with parameterised `?`
placeholders. Add `err` inspection at every `db.run`, `db.get`, and `db.all`
callback, responding with appropriate HTTP status codes instead of failing
silently.

**Bob produced:**

All 7 runtime query sites converted. Error handling strategy per site:

| Route | Query | On error |
|---|---|---|
| `POST /api/shorten` | INSERT | `SQLITE_CONSTRAINT` → 409; other → 500 |
| `GET /:code` SELECT | SELECT | 500 |
| `GET /:code` UPDATE clicks | UPDATE | log + continue redirect (best-effort counter) |
| `GET /api/urls` | SELECT ALL | 500 |
| `GET /api/urls/:code` | SELECT | 500 |
| `PUT /api/urls/:code` | UPDATE | 500 |
| `DELETE /api/urls/:code` | DELETE | 500 |

The click-counter UPDATE uses a deliberate **log-and-continue** policy: a
counter failure should never block the user from reaching their destination.

---

### Prompt 2.6 — Modular Architecture and Test Suite

**Task given to Bob:**  
Decompose the single-file application into a clean module tree. Add an
integration test suite using Node's native `node:test` runner. Update
`package.json` with a `test` script.

**Bob produced:** The structure described in the next section, plus 21
integration tests across 5 suites (all passing).

---

## Resulting Architecture

```
server.js                        Production entry point (listen only)
src/
  config.js                      All constants + env overrides
  db.js                          SQLite connection + initSchema()
  app.js                         Express factory — no listen() call
  utils/
    validator.js                 validateUrl(), validateCode()
    shortener.js                 generateCode()
  controllers/
    urlController.js             All 7 handler functions
  routes/
    urlRoutes.js                 Router — paths bound to handlers
tests/
  urls.test.js                   21 integration tests (node:test)
docs/
  code-smells.md                 Original smell audit
  member-2-refactoring.md        This document
  architecture.svg               System architecture diagram
```

### Module responsibilities

| Module | Responsibility | Dependencies |
|---|---|---|
| `src/config.js` | Single source of truth for all constants | none |
| `src/db.js` | DB connection; `initSchema(done)` | `config` |
| `src/utils/validator.js` | Pure URL and code validation | `config` |
| `src/utils/shortener.js` | Random Base62 code generation | `config` |
| `src/controllers/urlController.js` | Request handlers / business logic | `db`, `validator`, `shortener`, `config` |
| `src/routes/urlRoutes.js` | Express Router bindings | `controller` |
| `src/app.js` | Middleware + route mount; exports `app` | `db`, `routes` |
| `server.js` | `app.listen(PORT)` | `app`, `config` |

### Why `src/app.js` does not call `listen`

Separating the Express app from the listen call makes the application
**importable by tests** without binding a real port. The test suite calls
`http.createServer(app).listen(0)` — port `0` means the OS assigns an
ephemeral port — and closes the server cleanly after each suite. This
eliminates port conflicts and makes each test suite hermetically isolated
against a fresh `:memory:` SQLite database.

---

## Test Suite

**Run with:**
```bash
npm test
# node --test tests/urls.test.js
```

**Coverage (21 tests, 5 suites):**

| Suite | Tests |
|---|---|
| `POST /api/shorten` | auto-code, custom-code, 409 duplicate, 400 missing url, bad protocol, code too short/long |
| `GET /:code` (redirect) | 302 + Location header, 404 unknown, 400 code too short |
| `PUT /api/urls/:code` | 200 update, redirect follows new URL, 404 unknown, 400 invalid URL |
| `DELETE /api/urls/:code` | 200 delete, post-delete 404, 404 unknown |
| `GET /api/urls` + `GET /api/urls/:code` | array shape, newest-first order, `short_url` field, single fetch 200/404 |

---

## Bob Session Screenshots

Screenshots of each Bob interaction are in
[`bob_sessions/partho/`](../bob_sessions/partho/):

| File | Content |
|---|---|
| `01_smell_audit.png` | Prompt 2.1 — smell catalogue output |
| `02_refactor_validation.png` | Prompt 2.2 — `validateUrl()` extraction |
| `03_refactor_guard_clauses.png` | Prompt 2.3 — guard-clause flattening |
| `04_refactor_constants_naming.png` | Prompt 2.4 — constants and renaming |
| `05_refactor_sql_security.png` | Prompt 2.5 — parameterised SQL + error handling |
| `06_modular_tests.png` | Prompt 2.6 — modular architecture and test suite |
