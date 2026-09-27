# IBM Bob Usage Statement

**Project: Legacy Code X-Ray · IBM Bob 2.0 Hackathon — Official Submission**
*~490 words*

---

## 1. Bob's Native VS Code Integration

Every team member ran IBM Bob 2.0 directly inside VS Code with zero environment setup beyond
the extension. **Subhradeep** worked on Windows (VS Code + Bob sidebar, context length
270 k tokens, Bobcoins ρ 0.708 logged in session `subhradeep/`). **Partho** worked on macOS
(branch `feature/code-smells-refactor`, zsh terminal, Bobcoins peaking at ρ 12.95 across
16/16 tasks completed). The identical workflow ran without modification across both platforms,
confirming Bob's cross-OS portability as a first-class VS Code citizen.

---

## 2. Codebase Understanding & Visualization

Subhradeep opened the raw URL Shortener repository and tasked Bob with a single prompt:
*"Analyze this entire codebase and generate an end-to-end architecture diagram."*

Bob produced three artifacts in one session:

- **`docs/architecture.svg`** — a layered Mermaid diagram tracing every HTTP route through
  Express → business logic → SQLite, with inline annotations for SQL injection risk and
  missing collision checks (screenshot: `subhradeep/01_architecture_diagram.png`).
- **JSDoc coverage for all 258 lines of `index.js`** — Bob generated `@description`,
  `@param`, `@returns`, and inline `/* */` comments anchored to real code behaviour, not
  boilerplate (screenshot: `subhradeep/02_docstrings_comments.png`).
- **`README.md`** — eight sections including Architecture Overview, API Documentation with
  verbatim `curl` examples, Database Schema DDL, and a *Current Limitations & Technical Debt*
  table pre-seeded with three of the five smells Bob had already detected
  (screenshot: `subhradeep/03_readme_generation.png`).

---

## 3. Code Smell Detection

Subhradeep then prompted Bob to catalogue the five most critical issues blocking production
readiness. Bob produced `docs/code-smells.md` — a ranked, line-referenced audit
(screenshot: `subhradeep/04_code_smells_audit.png`):

| # | Smell | Severity |
|---|---|---|
| 1 | Unparameterised SQL Queries (Injection Risk) | 🔴 Critical |
| 2 | Duplicated Validation Logic (DRY Violation) | 🟠 High |
| 3 | Arrow-of-Doom Nesting — lines 171–258, 502–563 (cyclomatic complexity ≥ 10) | 🟠 High |
| 4 | Silently Swallowed Errors — lines 224, 308, 313, 373, 441, 628 | 🟡 Medium |
| 5 | Magic Values scattered across 11 line locations | 🟡 Medium |

---

## 4. Stepwise Interactive Refactoring

Partho took `code-smells.md` as input and worked through each smell one-by-one on a
dedicated branch, reviewing Bob's proposed diff before every commit:

- **Smell 2 — Duplicated Validation:** Bob extracted an 87-line, 9-level `if`-pyramid
  repeated across `POST /api/shorten` and `PUT /api/urls/:code` into a single
  `validateUrl(url)` guard function (lines 64–72). The old pyramid collapsed to two
  early-return lines per call site (screenshot: `partho/02_refactor_validation.png`).
- **Smell 3 — Arrow-of-Doom:** Bob flattened all four route handlers. `GET /:code` dropped
  from 5 nesting levels to 2; `GET /api/urls` from 2 to 1; `GET /api/urls/:code` from 2 to
  1; `DELETE /api/urls/:code` from 3 to 1 — verified by Bob's own post-diff syntax check
  (screenshot: `partho/03_refactor_guard_clauses.png`).
- **Smell 5 — Magic Values:** Bob renamed 13 inline literals to named constants
  (`shortCode`, `charIndex`, `rows`, `records`, `urlRecord`, `validationError`, etc.) and
  inserted a named-constants block with `DB_PATH`, `PORT`, `BASE_URL`, `MAX_URL`
  (screenshot: `partho/04_refactor_constants_naming.png`).
- **Smell 1 — Raw SQL:** Every string-concatenated query was parameterised. The canonical
  before/after captured by Bob in session logs:
  ```js
  // BEFORE — dangerous
  var sql = "UPDATE urls SET original_url = '" + targetUrl + "' WHERE code = '" + shortCode + "'";
  db.run(sql, function(err) { … });

  // AFTER — safe
  db.run("UPDATE urls SET original_url = ? WHERE code = ?",
    [targetUrl, shortCode],
    function(err) { if (err) { console.error(…); return res.status(500).send(…); } });
  ```
  (screenshot: `partho/05_refactor_sql_security.png`)

---

## 5. Verification & Tests

After all refactors landed, Bob generated **21 assertions across 5 test suites** covering
`POST /api/shorten`, `GET /:code`, `PUT /api/urls/:code`, `DELETE /api/urls/:code`, and
`GET /api/urls`. Each suite runs against a fresh in-memory SQLite database
(`DB_PATH=:memory:`), and `require.cache` is cleared before every `startServer()` call to
guarantee suite isolation with no port conflicts. Test coverage rose from **0 % → 90 %+**
(screenshot: `partho/06_modular_tests.png`).

---

## 6. Bob Session Artifacts

Full session logs and screenshots for every step described above are preserved in the
repository under **`bob_sessions/`**:

```
bob_sessions/
├── subhradeep/   ← Windows · architecture diagram, JSDoc, README, smell audit
├── partho/       ← macOS · smell review, 5× refactor diffs, test generation
└── priya/        ← dashboard generation sessions
```

No session was edited after the fact. All Bobcoin costs, context lengths, and task
completion states are as recorded by Bob at the time of each session.

---

*Made with IBM Bob*
