# The Power of 10: Rules for Python Crypto Systems

These instructions govern Python code written or modified in this repository. They are inspired by Gerard J. Holzmann's *The Power of Ten: Rules for Developing Safety-Critical Code*, adapted for cryptocurrency, custody, payments, trading, and other systems where bugs can lose funds.

## Requirement levels

- **MUST** / **MUST NOT**: Blocking requirement. The agent must satisfy it for any newly written or materially changed code in scope. If compliance is impossible, do not silently bypass it: explain the conflict and ask for an explicit decision before proceeding with the unsafe portion.
- **SHOULD** / **SHOULD NOT**: Strong default. An exception is acceptable when the agent can identify a concrete technical reason, preserve the relevant safety invariant, and briefly document the tradeoff.
- **Scope:** These rules govern newly written or materially modified Python code. Do not rewrite unrelated legacy code solely to enforce them. When modifying existing unsafe code, do not extend or rely on its unsafe behavior; flag relevant inherited risks.
- **Precedence:** Follow applicable higher-priority instructions and established project conventions where safe. If a convention conflicts with a MUST safety requirement, flag the conflict instead of weakening the requirement.

**Prime directive:** Optimize for correctness, clarity, auditability, and safe recovery—not cleverness or minimal line count. Treat funds, credentials, signatures, and persisted financial state as safety-critical. If a requirement is ambiguous in a way that could affect money or security, ask before implementing. Do not silently relax these rules.

## 1. Keep control flow simple and bounded

- **SHOULD:** Prefer explicit, linear logic with clear entry and exit conditions. Avoid clever metaprogramming, implicit side effects, and unnecessary recursion.
- **MUST:** Every loop, retry, polling operation, and batch must have a documented maximum or a clear external termination condition enforced by a timeout/cancellation mechanism.
- **MUST:** Use explicit timeouts, bounded concurrency, and backoff with jitter for network calls. Never retry indefinitely.
- **SHOULD:** Avoid hidden I/O in constructors, properties, or import-time code.

## 2. Represent money and blockchain quantities exactly

- **MUST:** Never use `float` for balances, prices used in settlement, fees, token quantities, or ledger entries. Use integer atomic units or `Decimal` with explicitly defined precision and rounding.
- **MUST:** Never silently mix wei, gwei, satoshis, token atomic units, human-readable token amounts, or different currencies/assets. Make units and asset identifiers explicit in names or types.
- **MUST:** Convert amounts at well-defined boundaries. Validate token decimals, rounding policy, numeric bounds, and dust/minimum amounts before any transfer or posting.
- **MUST:** Never assume all ERC-20 tokens use 18 decimals or that every asset shares the same precision.

## 3. Write small, typed, documented functions

- **SHOULD:** Prefer functions that perform one responsibility. Aim for no more than ~60 lines per function; split longer functions when it improves comprehension.
- **MUST:** Add type annotations to public functions and financial-domain interfaces. Prefer structured models, enums, and value objects over ambiguous dictionaries and booleans.
- **MUST:** Every new or substantially changed function must have a concise docstring explaining its purpose; document units, assumptions, side effects, and exceptional behavior when relevant. Use comments for non-obvious *why*, not obvious *what*.
- **MUST:** Keep mutable state narrowly scoped. Avoid mutable global state and mutable default arguments.

## 4. Validate every trust boundary

- **MUST:** Treat API requests, RPC responses, webhooks, event logs, environment variables, database records from external sources, and third-party SDK output as untrusted.
- **MUST:** Validate schemas and semantic constraints: addresses, chain IDs, asset IDs, nonnegative amounts, decimals, timestamps, block numbers, lengths, and allowed states.
- **MUST:** Verify authorization, account ownership, and destination allowlists/policies *before* constructing, signing, or submitting transfers.
- **MUST:** Reject missing, malformed, contradictory, stale, or unsupported inputs explicitly. Do not invent defaults for chain, asset, amount, destination, or signing configuration.

## 5. Make failures explicit and fail closed

- **MUST:** Never swallow exceptions, use bare `except:`, or convert uncertain outcomes into success. Catch specific exceptions at boundaries and preserve their context.
- **MUST:** Distinguish **rejected**, **failed**, **submitted**, **pending**, **confirmed**, and **unknown** transaction outcomes. A network timeout after broadcast is **unknown**, not proof of failure.
- **MUST:** Do not automatically resubmit a transaction or issue a compensating transfer until its prior outcome has been reconciled.
- **MUST:** Use runtime validation and domain-specific exceptions for externally supplied data. Do not use Python `assert` for production-critical validation because optimized execution can disable assertions.

## 6. Make money-moving operations idempotent

- **MUST:** Every externally initiated financial operation must have a stable, unique idempotency key or equivalent durable uniqueness constraint.
- **MUST:** Enforce deduplication at the database/transactional boundary, not only with in-memory checks.
- **MUST:** Record intent and state transitions durably before or alongside irreversible side effects; use an outbox or equivalent recoverable orchestration where appropriate.
- **MUST:** Retries must reconcile the original attempt first. Never assume delivery is exactly once across networks, queues, RPC providers, or webhooks.

## 7. Control concurrency and preserve financial invariants

- **MUST:** Define invariants for balances, reservations, ledger postings, nonce allocation, and transaction state transitions. Make invalid transitions impossible or explicitly rejected.
- **MUST:** Use database transactions, unique constraints, conditional updates, and appropriate locking/isolation to prevent races and double spending.
- **MUST:** Use double-entry ledger principles where accounting for value: entries must balance within the applicable asset and accounting model, and posted entries should be immutable with explicit reversals/corrections.
- **MUST:** Treat blockchain reorgs, delayed confirmations, duplicate/out-of-order events, and process crashes as expected scenarios; make reconciliation repeatable.

## 8. Keep secrets and signing isolated

- **MUST:** Never hardcode, print, log, commit, or expose private keys, seed phrases, API secrets, or raw signing credentials.
- **SHOULD:** Prefer managed signing (HSM/KMS/MPC or an established custody provider) and least-privilege access. Keep the signing boundary separate from application business logic where feasible.
- **MUST:** Use established, maintained cryptographic libraries. Never implement custom cryptographic primitives or invent a signing format.
- **MUST:** Before signing, verify the exact chain/domain, recipient, asset, amount, contract method, nonce, fee policy, and authorization. Protect against replay and chain/domain confusion.

## 9. Assume external systems and chains are unreliable

- **SHOULD:** Treat RPC providers, custodians, exchanges, price feeds, and bridge services as fallible. Validate responses and do not trust a single service's success flag as proof of settlement.
- **MUST:** Explicitly model chain finality and required confirmations by chain and operation; detect and reconcile reorgs rather than assuming that inclusion is final.
- **MUST:** Validate contract addresses and ABIs from trusted configuration. For high-risk operations, simulate or otherwise validate transaction effects where supported; reject unexpected effects.
- **MUST:** Apply bounded rate limits, exposure/velocity limits, circuit breakers, and human approval thresholds according to the risk of the operation. Do not bypass these controls for convenience.

## 10. Make correctness demonstrable before shipping

- **SHOULD:** For all changed Python code, run available formatting, linting, type checking, and tests; prefer Ruff, a type checker (Pyright or mypy), and pytest when the project has no established alternatives.
- **MUST:** Financial and signing code needs tests for normal cases, boundary values, precision/rounding, duplicates, retries, timeout-after-submit, crashes, races, reorgs, and failed dependencies.
- **SHOULD:** Use property-based tests or invariant tests where valuable, especially for amount conversion, ledger balancing, and state transitions.
- **MUST:** Never use real keys or live funds in tests. Prefer mocks, fixtures, testnets, or local forks, while recognizing their limits.
- **MUST:** Do not claim a change is safe or verified unless checks actually ran. Report what was tested, what was not, and any remaining risk.

## Compliance and exception handling

- For a **MUST** rule, do not claim compliance unless the implemented code meets the rule. When blocked by unclear product/security requirements, stop work on the affected operation and ask a targeted question. An explicit exception request must be reported as a safety deviation; it does not turn unsafe behavior into a verified safe design.
- For a **SHOULD** rule, prefer compliance. If deviating, state the specific reason and mitigation in the change summary; do not burden routine changes with unnecessary commentary when all defaults are met.
- Existing safety issues outside the requested change should be reported separately rather than silently refactored.
- In the final response for high-risk changes, provide a concise **Safety checklist**: applicable MUST requirements, any SHOULD deviations, verification performed, and outstanding risks.

## Agent execution protocol

When implementing or reviewing a change:

1. **Identify risk:** Is this code involved in balances, ledger state, order execution, withdrawals, custody, signing, permissions, or blockchain finality? If yes, treat it as high risk.
2. **State invariants:** For high-risk changes, identify the critical invariants, expected failure modes, and idempotency/reconciliation strategy before writing code.
3. **Prefer minimal changes:** Preserve existing architecture and established project conventions unless they conflict with these safety requirements. Do not make unrelated refactors.
4. **Write the implementation:** Keep operations typed, documented, observable, and recoverable. Never add unsafe fallback behavior just to make an operation pass.
5. **Verify:** Run relevant checks and focused negative-path tests. If checks cannot run, explicitly say why.
6. **Report:** Summarize changed files, enforced invariants, tests run, and unresolved safety concerns. Do not describe unverified behavior as guaranteed.

**Escalation rule:** If a requested implementation could lose, duplicate, misdirect, freeze, or expose funds or keys, and a safety-critical requirement is unspecified, stop and ask a targeted question before making that decision.
