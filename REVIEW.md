# ExpenseFlow — Production Handoff Review

Reviewed the full `app/` package, `docs/ARCHITECTURE.md`, `.env`, and repo root against `CLAUDE.md`. Findings below, most severe first. Problems only — no suggested fixes.

## Critical

**1. Live API secret sitting unprotected in the repo, no `.gitignore` anywhere**
`.env` contains a real `ANTHROPIC_API_KEY` value. There is no `.gitignore` file in the entire project — nothing prevents a future `git add .` / `git add -A` from committing that key (and `expenseflow.db`, which holds real submitted data) straight into git history.
Why it matters: once a secret enters git history it's compromised permanently — rotation is mandatory, and history rewriting doesn't fully undo exposure if it's ever pushed anywhere.

**2. FX conversion is a hardcoded no-op**
`create_expense` (`app/routes.py:19-24`) always sets `amount_base_minor = payload.amount_minor` and `fx_rate_used = 1.0`, regardless of `currency`, with a `# TODO` comment acknowledging it's unfinished.
Why it matters: CLAUDE.md defines the entire product as "submit → convert to base currency → approve/reject." For any non-INR submission, the stored base amount is simply wrong (e.g. 100 USD gets recorded as ₹1.00). This is silent financial data corruption in the one workflow the app exists for, not a missing nice-to-have.

**3. Zero automated test coverage**
CLAUDE.md documents `python -m pytest -q` as the test command, and `docs/ARCHITECTURE.md` describes a `tests/test_expenses.py` suite covering conversion, rounding, and approve/reject transitions — none of it exists. No `tests/` directory is present anywhere in the project.
Why it matters: there is no automated verification of the approval state machine or any of the edge cases the architecture doc itself flags as risky. A handoff with no tests means every future change is unverified by anything but manual testing.

## High

**4. No authentication or authorization on any endpoint**
`submitted_by` and `decided_by` (`app/schemas.py`) are unauthenticated free-text strings. Any caller can approve or reject any expense and self-report as any name.
Why it matters: for a financial approval workflow, "approval" carries no actual access control — nothing enforces who is allowed to approve spend.

**5. An endpoint exists that the project's own spec says was deliberately excluded**
`GET /reports/insights` (`app/routes.py:90-98`, `app/insights.py`) is not in CLAUDE.md's brief, and `docs/ARCHITECTURE.md` Section 0 explicitly states the insight feature was excluded per the "do not invent endpoints not in the brief" rule. It exists anyway and sends internal expense records to the Anthropic API.
Why it matters: it contradicts the project's own written architecture decision and its "do not invent endpoints" rule, and it exfiltrates internal financial data to a third-party service with no mention in the endpoint table, no opt-out, and no data minimization.

**6. Hardcoded model identifier that doesn't match a known Anthropic model id**
`MODEL = "claude-sonnet-4-6"` (`app/insights.py:14`).
Why it matters: if this id is invalid, every call to `/reports/insights` fails at the API layer. The broad `except Exception` around it (line 82-83) swallows that failure and always returns the canned fallback, so the feature could be permanently non-functional while appearing "handled" in logs.

## Medium

**7. Race condition in approve/reject state transitions**
Both handlers (`app/routes.py:54-87`) do read → check `status == "pending"` → mutate → commit, with no row lock or transaction isolation.
Why it matters: two concurrent requests against the same expense id can both pass the pending check before either commits, defeating the exact one-way-transition guarantee `docs/ARCHITECTURE.md` Section 4 edge case 3 claims is enforced.

**8. Insight prompt references a field that doesn't exist on the data it's given**
`_summarize_expenses` (`app/insights.py:27`) reads `expense.get("category")`, but `Expense` (`app/models.py`) has no `category` column, and `get_insights` (`app/routes.py:94-97`) never puts one in the dict it builds.
Why it matters: every line fed into the LLM prompt literally contains the string "None" where a category should be, silently degrading insight quality with no error surfaced anywhere.

**9. Currency validation accepts nonsense codes**
`currency_must_be_three_letters` (`app/schemas.py:16-23`) only checks length and alphabetic characters — any 3-letter string like `"ZZZ"` passes.
Why it matters: combined with issue #2, an unrecognized currency is stored and "converted" at a rate of 1.0 with no rejection, producing a nonsensical base amount with no validation failure to flag it.

**10. Unbounded `GET /expenses`**
`list_expenses` (`app/routes.py:34-42`) always returns every row in the table, with no pagination or limit.
Why it matters: response size and query cost grow unbounded with the table, a straightforward availability/performance issue as data accumulates.

**11. No dependency manifest anywhere in the repo**
No `requirements.txt`, `pyproject.toml`, or lock file exists.
Why it matters: there is no reproducible, authoritative record of what versions this code was built and tested against for a production deploy to install.

**12. Reject "reason" is accepted by the API and silently dropped**
`ApproveRejectIn.reason` (`app/schemas.py:30`) has no backing database column; `reject_expense` (`app/routes.py:83-84`) accepts it and discards it without persisting it or indicating that in the response.
Why it matters: a caller supplying a rejection reason reasonably assumes it was recorded — it disappears with no error and no audit trail.

**13. Hardcoded, working-directory-relative database path**
`DATABASE_URL = "sqlite:///expenseflow.db"` (`app/db.py:13`) is a relative path with no environment-based override.
Why it matters: running the process from a different working directory silently creates or reads a different database file, and there is no way to target a different database per environment without editing source.

## Low

**14. Broad exception handling collapses distinct failure classes**
`generate_insight`'s retry loop (`app/insights.py:79-83`) catches bare `Exception`, treating a network timeout, a malformed API response, and a genuine code bug (e.g. an `AttributeError`) identically — logged and swallowed the same way.
Why it matters: logs alone can't distinguish "the external service had a bad day" from "we shipped a bug."

**15. No logging or metrics on the core workflow**
The only logging in the codebase is the exception log inside `insights.py`; `create_expense`, `approve_expense`, and `reject_expense` have none.
Why it matters: a production incident in the core expense workflow (e.g. a spike in 409/404 responses) would leave no trace to diagnose from.

**16. No constraint on `description` length or emptiness**
`ExpenseCreate.description` (`app/schemas.py:11`) accepts any string, including empty or arbitrarily large ones.
Why it matters: nothing prevents an empty description on a financial record or a very large string being persisted per row.
