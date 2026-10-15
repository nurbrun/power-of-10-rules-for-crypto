# The Power of 10 for Python Crypto Systems

Inspired by Gerard J. Holzmann's *The Power of Ten: Rules for Developing Safety-Critical Code* (2006). These rules govern new and materially modified Python code in this repository, especially code handling funds, balances, transactions, keys, and signing.

**Goal:** Code must be simple enough to review, bounded enough to operate safely, and precise enough to verify. Optimize for correctness and auditability over cleverness.

**MUST** means a blocking requirement; **SHOULD** means the default unless a concrete reason justifies an exception. Do not silently waive a MUST. For a safety-critical conflict or missing policy, stop the affected work and ask. Do not refactor unrelated legacy code merely to conform; flag relevant inherited risks.

## 1. Use simple, explicit control flow

- **MUST:** Make state transitions and side effects visible. Reject invalid transitions rather than relying on implicit behavior.
- **SHOULD:** Prefer straightforward branches and iteration; avoid recursion, metaprogramming, dynamic dispatch, and hidden I/O where explicit code is clearer.

## 2. Bound every operation

- **MUST:** Give loops, retries, queues/batches, network calls, and concurrent work explicit limits (iterations, size, timeout, or cancellation/deadline as appropriate). Never retry or poll indefinitely.
- **MUST:** Bound financial exposure with applicable amount, fee, rate, and velocity limits. A timeout is not evidence that a submitted transaction failed.

## 3. Make quantities and resources explicit

- **MUST:** Represent financial quantities using integer atomic units or `Decimal` with specified precision and rounding; never use `float` for money or settlement calculations.
- **MUST:** Carry asset, chain, and unit identifiers across interfaces. Check numeric bounds and token decimals; never assume all tokens have 18 decimals.
- **SHOULD:** Bound memory and resource consumption by limiting collections, payloads, and in-flight tasks rather than materializing unbounded data.

## 4. Keep functions small and responsibilities narrow

- **SHOULD:** Keep functions to one responsibility and roughly 60 lines or fewer; break them up when doing so makes reasoning easier.
- **MUST:** Give each new or materially modified function a concise docstring describing its purpose; document non-obvious units, side effects, and failure behavior. Comment *why* rather than narrating *what*.

## 5. State and check invariants

- **MUST:** Define and enforce relevant invariants: e.g., no negative withdrawable balance, balanced ledger postings, unique nonce allocation, valid state transitions, authorized destinations.
- **MUST:** Check invariants with executable validations or tests; fail closed on violations. Never use Python `assert` for production input or financial validation (`-O` can remove it).
- **SHOULD:** Prefer assertions in tests and focused invariant/property-based tests over redundant defensive checks that cannot fail.

## 6. Minimize mutable state and authority

- **MUST:** Scope mutable state, credentials, and permissions to the narrowest practical component; avoid mutable globals and shared unsynchronized state.
- **MUST:** Never hardcode, log, commit, or expose private keys or secrets. Separate signing authority from ordinary business logic and verify the transaction intent before signing.
- **SHOULD:** Use established signing libraries and managed key custody; never devise custom cryptographic primitives.

## 7. Validate inputs and handle every outcome

- **MUST:** Validate all trust-boundary inputs (API, RPC, webhook, database imports, configuration, SDK responses), including schema, authorization, chain, asset, amount, and destination.
- **MUST:** Handle failure and uncertainty explicitly; never swallow exceptions or interpret an RPC timeout as a definitive failure. Distinguish rejected, submitted, pending, confirmed, and unknown outcomes where relevant.
- **MUST:** Check consequential return values and external responses; do not silently substitute defaults for critical transaction fields.

## 8. Keep critical behavior inspectable

- **MUST:** Make money movement and signing paths traceable in ordinary code. Do not conceal side effects in decorators, dynamic imports, reflection, monkey-patching, or opaque helpers.
- **MUST:** Use vetted cryptographic and protocol libraries; validate trusted contract/chain configuration and signing payloads (chain/domain, recipient, asset, amount, nonce, fees) before use.
- **SHOULD:** Favor explicit types, typed domain objects, and a small number of clear abstractions over generic dictionaries and clever frameworks.

## 9. Design for replay, concurrency, and recovery

- **MUST:** Give externally initiated financial actions durable idempotency protection; enforce uniqueness transactionally, not only in memory. Reconcile uncertain outcomes before resubmitting or compensating.
- **MUST:** Prevent balance and nonce races using appropriate database transactions, constraints, or locking. Persist intent and state transitions so crashes are recoverable.
- **MUST:** Handle duplicate/out-of-order events, reorgs, and chain-specific finality before treating settlement as final. Ledger postings must balance and corrections must remain auditable.

## 10. Verify continuously and report evidence

- **MUST:** Run the project's formatter/linter, type checker, and relevant tests when available; introduce or run strong static checks for financial paths. Do not knowingly leave new warnings or failing checks unexplained.
- **MUST:** Test safety-critical paths for boundary values, precision, invalid inputs, duplicate requests, races, timeouts after submit, crashes, failed dependencies, and chain reorgs where applicable. Never test with real keys or live funds.
- **MUST:** Report which checks actually ran, what could not be verified, and any unresolved safety deviations. Never claim unrun tests passed.

## Agent workflow (not additional rules)

1. Identify whether the change touches funds, custody, signing, ledger state, authorization, or finality.
2. For such changes, state the relevant invariants and uncertain outcomes before implementing.
3. Make the smallest safe change and verify it against the ten rules.
4. Report tests/checks performed and outstanding risk, briefly. Explain any **SHOULD** exception; pause and request direction if a **MUST** cannot be met.

Source: https://en.wikipedia.org/wiki/The_Power_of_10:_Rules_for_Developing_Safety-Critical_Code
