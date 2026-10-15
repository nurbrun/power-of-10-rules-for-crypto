# Python — The Power of 10 for Crypto Systems

**Ten engineering principles for writing safer, more predictable, and auditable Python code in cryptocurrency systems.**

This is a Python-specific adaptation of Gerard J. Holzmann's [*The Power of Ten: Rules for Developing Safety-Critical Code*](https://en.wikipedia.org/wiki/The_Power_of_10:_Rules_for_Developing_Safety-Critical_Code), with particular attention to financial precision, transaction integrity, cryptographic operations, and failure recovery.

## Getting started

Copy [`AGENTS.md`](AGENTS.md) into the root of your Python project to provide these instructions to Cursor or another compatible AI coding agent.

```text
your-project/
├── AGENTS.md
├── pyproject.toml
├── src/
└── tests/
```

## Reading the rules

Each of the ten principles contains individual **MUST** and **SHOULD** requirements, illustrated with practical Python examples. Each sub-rule reproduces its corresponding `AGENTS.md` clause verbatim for one-to-one traceability.

| Section | Purpose |
|---|---|
| **Violation** | Code that breaks the requirement |
| **Risk** | What could go wrong |
| **Compliance** | Code demonstrating the safer approach |
| **Why this is safer** | The safety property gained |
| **Trade-offs** | Performance, complexity, and operational costs |

### Requirement levels

- **MUST** — A blocking requirement that must not be silently bypassed.
- **SHOULD** — A strong default; deviations require concrete technical justification.

Examples are intentionally simplified to isolate individual engineering principles and should not be treated as complete, production-ready implementations.

## The Ten Rules

1. **Use simple, explicit control flow**
2. **Bound every operation**
3. **Make quantities and resources explicit**
4. **Keep functions small and responsibilities narrow**
5. **State and check invariants**
6. **Minimize mutable state and authority**
7. **Validate inputs and handle every outcome**
8. **Keep critical behavior inspectable**
9. **Design for replay, concurrency, and recovery**
10. **Verify continuously and report evidence**

---

## 1. Use simple, explicit control flow

### 1.1 MUST — Make transitions and side effects visible

> **AGENTS.md requirement:** Make state transitions and side effects visible. Reject invalid transitions rather than relying on implicit behavior.

**Violation**

```python
# Unknown states silently become confirmed.
next_state = {"pending": "confirmed"}.get(tx.state, "confirmed")
credit_account(tx)
```

**Risk:** Implicit transitions can credit funds from an invalid state.

**Compliance**

```python
if tx.state != "pending":
    raise InvalidTransition(tx.state)
tx.state = "confirmed"
credit_account(tx)
```

**Why this is safer:** Explicit state checks reject invalid transitions before money moves.

**Trade-offs:** Extra branches and validation add a little code and CPU work; for transfers, that cost is negligible compared with the risk of an invalid transition.

### 1.2 SHOULD — Prefer explicit branches over clever indirection

> **AGENTS.md requirement:** Prefer straightforward branches and iteration; avoid recursion, metaprogramming, dynamic dispatch, and hidden I/O where explicit code is clearer.

**Violation**

```python
# Dynamic dispatch hides which money-moving function runs.
getattr(wallet, request.action)(request.amount)
```

**Risk:** Dynamic control flow makes privileged actions harder to audit and constrain.

**Compliance**

```python
if request.action == "deposit":
    wallet.deposit(request.amount)
elif request.action == "withdraw":
    wallet.withdraw(request.amount)
else:
    raise ValueError("Unsupported action")
```

**Why this is safer:** Named branches make allowed actions obvious and unexpected actions fail clearly.

**Trade-offs:** Explicit dispatch grows linearly with the number of supported actions and may require edits when adding one; dispatch overhead is typically negligible.

## 2. Bound every operation

### 2.1 MUST — Limit work, retries, and waiting

> **AGENTS.md requirement:** Give loops, retries, queues/batches, network calls, and concurrent work explicit limits (iterations, size, timeout, or cancellation/deadline as appropriate). Never retry or poll indefinitely.

**Violation**

```python
while True:
    try:
        return rpc.get_receipt(tx_hash)
    except RpcUnavailable:
        pass
```

**Risk:** Unbounded loops or I/O can exhaust workers and prevent recovery.

**Compliance**

```python
for attempt in range(5):
    try:
        return rpc.get_receipt(tx_hash, timeout=3)
    except RpcUnavailable:
        if attempt == 4:
            raise
raise RuntimeError("Unreachable")
```

**Why this is safer:** Finite retries and timeouts cap resource use and surface persistent failures.

**Trade-offs:** Timeouts and retry caps can abandon operations during temporary outages, while repeated attempts still consume latency and capacity; use reconciliation or later scheduled recovery rather than unbounded waiting.

### 2.2 MUST — Limit financial exposure and preserve submission uncertainty

> **AGENTS.md requirement:** Bound financial exposure with applicable amount, fee, rate, and velocity limits. A timeout is not evidence that a submitted transaction failed.

**Violation**

```python
try:
    rpc.broadcast(signed_tx)
except TimeoutError:
    rpc.broadcast(sign_new_transaction(transfer))
```

**Risk:** A timeout does not cancel a broadcast; blindly sending again may spend twice or exceed limits.

**Compliance**

```python
limits.require_allowed(transfer.amount, transfer.fee, transfer.account_id)
try:
    rpc.broadcast(signed_tx, timeout=5)
except TimeoutError:
    store.mark_outcome_unknown(transfer.id)
    raise SubmissionOutcomeUnknown(transfer.id)
```

**Why this is safer:** Exposure checks and an explicit unknown state prevent unsafe resubmission after timeouts.

**Trade-offs:** Persisting uncertain outcomes and checking limits add storage I/O and may delay completion until reconciliation; avoid treating that latency as permission to issue a fresh transfer.

## 3. Make quantities and resources explicit

### 3.1 MUST — Use exact monetary arithmetic

> **AGENTS.md requirement:** Represent financial quantities using integer atomic units or `Decimal` with specified precision and rounding; never use `float` for money or settlement calculations.

**Violation**

```python
fee = 0.1 + 0.2  # Binary float is inexact.
```

**Risk:** Floating-point rounding can misprice transfers or break reconciliation.

**Compliance**

```python
from decimal import Decimal, ROUND_UP
fee = (Decimal("0.1") + Decimal("0.2")).quantize(
    Decimal("0.01"), rounding=ROUND_UP
)
```

**Why this is safer:** Decimal arithmetic makes fees reproducible under an explicit rounding policy.

**Trade-offs:** `Decimal` and quantization are slower than native floating-point arithmetic; integer atomic units are often faster when the asset scale is known, and either exact approach is preferable for money.

### 3.2 MUST — Preserve chain, asset, units, decimals, and numeric bounds

> **AGENTS.md requirement:** Carry asset, chain, and unit identifiers across interfaces. Check numeric bounds and token decimals; never assume all tokens have 18 decimals.

**Violation**

```python
amount_atomic = int(request.amount * 10**18)  # Assumes 18 decimals.
send(request.destination, amount_atomic)
```

**Risk:** Wrong chain or decimal assumptions can transfer the wrong asset or the wrong amount.

**Compliance**

```python
from decimal import Decimal
asset = registry.require_asset(request.chain_id, request.asset_id)
amount = Decimal(request.amount)
amount_atomic = asset.to_atomic_exact(amount)  # Validates scale and bounds.
send(chain_id=asset.chain_id, asset_id=asset.id,
     destination=request.destination, amount_atomic=amount_atomic)
```

**Why this is safer:** Asset metadata and exact conversion prevent unit mistakes before transaction submission.

**Trade-offs:** Metadata lookups and strict conversions add validation cost and can reject formerly accepted inputs; cache trusted immutable asset metadata where appropriate without bypassing validation.

### 3.3 SHOULD — Bound memory and in-flight resources

> **AGENTS.md requirement:** Bound memory and resource consumption by limiting collections, payloads, and in-flight tasks rather than materializing unbounded data.

**Violation**

```python
transactions = list(db.stream_all_transactions())
results = await asyncio.gather(*(inspect(tx) for tx in transactions))
```

**Risk:** Unbounded materialization or fan-out can exhaust memory and overwhelm dependencies.

**Compliance**

```python
semaphore = asyncio.Semaphore(16)

async def limited_inspect(tx):
    async with semaphore:
        return await inspect(tx)

for batch in db.iter_transaction_batches(size=100):
    await asyncio.gather(*(limited_inspect(tx) for tx in batch))
```

**Why this is safer:** Batching and concurrency limits keep memory use and dependency load predictable.

**Trade-offs:** Batching may increase individual item latency and semaphore limits may reduce peak throughput; tune limits using memory, RPC quotas, and observed load rather than maximizing concurrency blindly.

## 4. Keep functions small and responsibilities narrow

### 4.1 SHOULD — Give each function one responsibility

> **AGENTS.md requirement:** Keep functions to one responsibility and roughly 60 lines or fewer; break them up when doing so makes reasoning easier.

**Violation**

```python
def withdraw(request):
    validate(request)
    debit_balance(request)
    sign_transaction(request)
    broadcast_transaction(request)
    email_customer(request)
```

**Risk:** Combining distinct responsibilities makes failure paths and partial completion harder to reason about.

**Compliance**

```python
def prepare_withdrawal(request):
    """Validate and persist the requested withdrawal intent."""
    validated = validate_withdrawal(request)
    return persist_withdrawal_intent(validated)

# Separate components own debit, signing, submission, and notification.
```

**Why this is safer:** Separating intent creation from execution makes each failure boundary easier to test and recover.

**Trade-offs:** More functions and interfaces increase navigation and coordination overhead; avoid splitting trivial pure operations when the separation adds no meaningful failure boundary.

### 4.2 MUST — Document function purpose and non-obvious contracts

> **AGENTS.md requirement:** Give each new or materially modified function a concise docstring describing its purpose; document non-obvious units, side effects, and failure behavior. Comment *why* rather than narrating *what*.

**Violation**

```python
def calculate_fee(v, r):
    return v * r // 10_000
```

**Risk:** Unstated units and rounding rules invite integration errors in financial calculations.

**Compliance**

```python
def calculate_fee(amount_atomic: int, rate_bps: int) -> int:
    """Return a fee in atomic units, rounded down from basis points."""
    return amount_atomic * rate_bps // 10_000
```

**Why this is safer:** A typed signature and docstring make units, purpose, and rounding behavior clear to callers.

**Trade-offs:** Docstrings and type annotations take effort to maintain and can become stale; they have little runtime overhead in ordinary use and should document contracts rather than repeat obvious code.

## 5. State and check invariants

### 5.1 MUST — Define and enforce financial/state invariants

> **AGENTS.md requirement:** Define and enforce relevant invariants: e.g., no negative withdrawable balance, balanced ledger postings, unique nonce allocation, valid state transitions, authorized destinations.

**Violation**

```python
balance.available_atomic -= withdrawal.amount_atomic
```

**Risk:** Unchecked invariants can permit negative balances or invalid state transitions.

**Compliance**

```python
if withdrawal.amount_atomic <= 0:
    raise InvalidAmount()
if balance.available_atomic < withdrawal.amount_atomic:
    raise InsufficientFunds()
balance.available_atomic -= withdrawal.amount_atomic
```

**Why this is safer:** Precondition checks reject invalid withdrawals before balances change.

**Trade-offs:** Runtime validation adds branches and sometimes database coordination; it can introduce contention on hot balances, so enforce invariants atomically at the storage boundary as well.

### 5.2 MUST — Enforce invariants at runtime; do not rely on `assert`

> **AGENTS.md requirement:** Check invariants with executable validations or tests; fail closed on violations. Never use Python `assert` for production input or financial validation (`-O` can remove it).

**Violation**

```python
assert request.amount_atomic > 0  # Disabled by python -O.
submit_withdrawal(request)
```

**Risk:** Python optimization can remove assertions and leave financial validation unenforced.

**Compliance**

```python
if request.amount_atomic <= 0:
    raise InvalidAmount("amount_atomic must be positive")
submit_withdrawal(request)
```

**Why this is safer:** Explicit exceptions keep critical validation active even with Python optimizations enabled.

**Trade-offs:** Explicit exceptions add small execution overhead on invalid paths and require callers to handle failures; expected high-frequency rejections may warrant a measured alternative that still enforces the rule.

### 5.3 SHOULD — Use focused invariant/property tests, not redundant guards

> **AGENTS.md requirement:** Prefer assertions in tests and focused invariant/property-based tests over redundant defensive checks that cannot fail.

**Violation**

```python
# A redundant defensive check obscures the real invariant.
if amount_atomic != amount_atomic:
    raise ValueError("invalid integer")
```

**Risk:** Meaningless checks add noise while leaving real accounting properties untested.

**Compliance**

```python
from hypothesis import given, strategies as st

@given(
    st.integers(min_value=0, max_value=10**12),
    st.integers(min_value=0, max_value=10**12),
)
def test_ledger_transfer_preserves_total(source, destination):
    ledger = InMemoryLedger(source=source, destination=destination)
    before = ledger.total_atomic()
    ledger.transfer_atomic(min(source, 7))
    assert ledger.total_atomic() == before
```

**Why this is safer:** Property tests exercise the ledger implementation and check its conservation invariant across many generated balances.

**Trade-offs:** Property-based tests increase test runtime and can require effort to reproduce failures; keep deterministic seeds/examples and distinguish test-only checks from runtime validation.

## 6. Minimize mutable state and authority

### 6.1 MUST — Minimize state scope and shared authority

> **AGENTS.md requirement:** Scope mutable state, credentials, and permissions to the narrowest practical component; avoid mutable globals and shared unsynchronized state.

**Violation**

```python
pending_withdrawals = []  # Shared mutable global.

def queue_withdrawal(w):
    pending_withdrawals.append(w)
```

**Risk:** Shared, unsynchronized mutable state creates races and disappears on process failure.

**Compliance**

```python
def queue_withdrawal(store, withdrawal):
    """Persist one withdrawal using the store's transactional interface."""
    with store.transaction() as tx:
        tx.insert_withdrawal(withdrawal)
```

**Why this is safer:** Transactional persistence narrows mutable state and preserves work across process failures.

**Trade-offs:** Transactions and durable storage add I/O, locking, and operational dependencies compared with a local list; they also enable multi-worker correctness and recovery.

### 6.2 MUST — Protect secrets; isolate and verify signing

> **AGENTS.md requirement:** Never hardcode, log, commit, or expose private keys or secrets. Separate signing authority from ordinary business logic and verify the transaction intent before signing.

**Violation**

```python
PRIVATE_KEY = "0x..."  # Embedded secret.
logger.info("Signing with %s", PRIVATE_KEY)
sign(PRIVATE_KEY, user_supplied_payload)
```

**Risk:** Exposed keys or unverified signing requests can allow irreversible asset theft.

**Compliance**

```python
intent = policy.validate_signing_intent(
    chain_id=transfer.chain_id,
    destination=transfer.destination,
    asset_id=transfer.asset_id,
    amount_atomic=transfer.amount_atomic,
)
signature = custody_signer.sign_approved_intent(intent)
```

**Why this is safer:** Policy-approved intents keep key access isolated from untrusted transaction requests.

**Trade-offs:** Policy evaluation and remote or hardware-backed signers add latency, cost, and availability dependencies; pre-validate requests and provision signer capacity without bypassing approval.

### 6.3 SHOULD — Use established signing and custody implementations

> **AGENTS.md requirement:** Use established signing libraries and managed key custody; never devise custom cryptographic primitives.

**Violation**

```python
def sign_transaction(message, private_key):
    return homemade_curve_math(message, private_key)
```

**Risk:** Homegrown cryptography can have subtle flaws that invalidate signatures or expose keys.

**Compliance**

```python
def sign_transaction(approved_intent, custody_signer):
    """Sign an already approved intent using managed custody."""
    return custody_signer.sign_approved_intent(approved_intent)
```

**Why this is safer:** Established custody implementations reduce custom cryptographic code that must be audited.

**Trade-offs:** Established custody tooling adds vendor dependencies, fees, and potentially network latency; weigh these against the audit and maintenance burden of bespoke cryptography.

## 7. Validate inputs and handle every outcome

### 7.1 MUST — Validate data at every trust boundary

> **AGENTS.md requirement:** Validate all trust-boundary inputs (API, RPC, webhook, database imports, configuration, SDK responses), including schema, authorization, chain, asset, amount, and destination.

**Violation**

```python
withdrawal = Withdrawal(**request.json())
execute_withdrawal(withdrawal)
```

**Risk:** Unvalidated input can bypass authorization or route funds to an unintended destination.

**Compliance**

```python
payload = WithdrawalRequest.model_validate(request.json())
user = authorize_withdrawal(request.auth, payload.account_id)
asset = registry.require_asset(payload.chain_id, payload.asset_id)
destination = validate_destination(asset.chain_id, payload.destination)
execute_withdrawal(user, asset, destination, payload.amount_atomic)
```

**Why this is safer:** Boundary validation ensures the caller, asset, and destination are checked before funds move.

**Trade-offs:** Schema and authorization checks add CPU and sometimes lookup latency, especially for remote policy/registry calls; cache only safe, appropriately scoped facts and recheck mutable permissions.

### 7.2 MUST — Represent failure and uncertainty explicitly

> **AGENTS.md requirement:** Handle failure and uncertainty explicitly; never swallow exceptions or interpret an RPC timeout as a definitive failure. Distinguish rejected, submitted, pending, confirmed, and unknown outcomes where relevant.

**Violation**

```python
try:
    broadcast(signed_tx)
except Exception:
    return "failed"
```

**Risk:** Treating an unknown outcome as a failure can trigger a duplicate payment.

**Compliance**

```python
try:
    tx_hash = broadcast(signed_tx, timeout=5)
except TimeoutError:
    return SubmissionResult(status="unknown")
except RpcRejected as exc:
    return SubmissionResult(status="rejected", reason=str(exc))
return SubmissionResult(status="submitted", tx_hash=tx_hash)
```

**Why this is safer:** Distinct outcomes let the caller reconcile uncertain submissions without treating them as failed.

**Trade-offs:** Tracking distinct outcomes adds states and reconciliation work, sometimes delaying user-visible completion; it avoids converting uncertainty into an unsafe retry.

### 7.3 MUST — Check consequential responses; reject missing critical fields

> **AGENTS.md requirement:** Check consequential return values and external responses; do not silently substitute defaults for critical transaction fields.

**Violation**

```python
chain_id = rpc_response.get("chainId", 1)
nonce = rpc_response.get("nonce", 0)
```

**Risk:** Silent defaults can sign for the wrong network or reuse a transaction nonce.

**Compliance**

```python
if "chainId" not in rpc_response or "nonce" not in rpc_response:
    raise InvalidRpcResponse("Missing chainId or nonce")
chain_id = validate_chain_id(rpc_response["chainId"])
nonce = validate_nonce(rpc_response["nonce"])
```

**Why this is safer:** Required-field validation stops malformed responses from silently changing transaction parameters.

**Trade-offs:** Strict response validation can reject partial or version-skewed RPC responses and require compatibility handling; the parsing cost is small relative to the cost of a bad nonce or chain ID.

## 8. Keep critical behavior inspectable

### 8.1 MUST — Keep signing and money movement visible

> **AGENTS.md requirement:** Make money movement and signing paths traceable in ordinary code. Do not conceal side effects in decorators, dynamic imports, reflection, monkey-patching, or opaque helpers.

**Violation**

```python
@autopay_on_read  # Decorator invisibly sends funds.
def get_balance(user_id):
    return ledger.balance(user_id)
```

**Risk:** Hidden side effects make it difficult to see when and why funds move.

**Compliance**

```python
def get_balance(user_id):
    """Read a balance without creating financial side effects."""
    return ledger.balance(user_id)

def submit_payout(approved_intent):
    """Submit an explicitly approved payout."""
    return payout_service.submit(approved_intent)
```

**Why this is safer:** Dedicated functions make balance reads and payout submission visibly distinct operations.

**Trade-offs:** Separating operations may require more calls or explicit orchestration, but makes side effects measurable and easier to isolate in tests.

### 8.2 MUST — Validate protocol, contract, and signing payload context

> **AGENTS.md requirement:** Use vetted cryptographic and protocol libraries; validate trusted contract/chain configuration and signing payloads (chain/domain, recipient, asset, amount, nonce, fees) before use.

**Violation**

```python
payload = build_transfer(to=address, value=amount)
signer.sign(payload)
```

**Risk:** Signing an unvalidated payload can authorize transfers on the wrong chain, to the wrong recipient, or with unsafe fees.

**Compliance**

```python
config = trusted_chains.require(chain_id)
intent = validate_transfer_intent(
    chain_id=config.chain_id,
    contract=config.allowed_token_contract,
    destination=address,
    asset_id=asset_id,
    amount_atomic=amount_atomic,
    nonce=reserved_nonce,
    max_fee_atomic=fee_limit,
)
signer.sign_approved_intent(intent)
```

**Why this is safer:** Context validation constrains what the signer is authorized to approve.

**Trade-offs:** Payload validation and policy lookups add CPU and may require RPC or data-store reads; use cached trusted configuration where safe, but validate dynamic chain state at the appropriate point.

### 8.3 SHOULD — Prefer typed domain objects and few clear abstractions

> **AGENTS.md requirement:** Favor explicit types, typed domain objects, and a small number of clear abstractions over generic dictionaries and clever frameworks.

**Violation**

```python
def send(tx: dict):
    rpc.send(tx["to"], tx["value"], tx["chain"])
```

**Risk:** Untyped generic structures make missing fields and unit confusion harder to detect.

**Compliance**

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class TransferIntent:
    chain_id: int
    destination: str
    amount_atomic: int

def send(tx: TransferIntent):
    rpc.send(tx.destination, tx.amount_atomic, tx.chain_id)
```

**Why this is safer:** Typed, immutable intent objects make transaction fields explicit and easier to validate.

**Trade-offs:** Typed objects introduce definitions and construction overhead versus loose dictionaries; in most transaction workflows that overhead is minor compared with network and signing latency.

## 9. Design for replay, concurrency, and recovery

### 9.1 MUST — Persist idempotency and reconcile uncertain submissions

> **AGENTS.md requirement:** Give externally initiated financial actions durable idempotency protection; enforce uniqueness transactionally, not only in memory. Reconcile uncertain outcomes before resubmitting or compensating.

**Violation**

```python
seen = set()
if request.id not in seen:
    seen.add(request.id)
    send_payment(request)
```

**Risk:** Process-local deduplication fails across restarts and workers, potentially paying twice.

**Compliance**

```python
# Database enforces UNIQUE(account_id, idempotency_key).
operation = store.get_or_create_payment_intent(
    account_id=request.account_id,
    idempotency_key=request.idempotency_key,
    details=request.details,
)
if operation.status == "unknown":
    return reconciler.resolve_before_retry(operation)
return payment_service.continue_once(operation)
```

**Why this is safer:** Durable idempotency records survive restarts, while reconciliation precedes any retry.

**Trade-offs:** Durable idempotency requires database writes, unique constraints, retained records, and reconciliation jobs; it can lower throughput or increase latency but is essential for safe retries.

### 9.2 MUST — Prevent races and persist recoverable state

> **AGENTS.md requirement:** Prevent balance and nonce races using appropriate database transactions, constraints, or locking. Persist intent and state transitions so crashes are recoverable.

**Violation**

```python
if db.get_balance(account_id) >= amount:
    db.set_balance(account_id, db.get_balance(account_id) - amount)
```

**Risk:** Concurrent requests can both pass a stale balance check and overspend an account.

**Compliance**

```python
with db.transaction() as tx:
    balance = tx.lock_account_balance(account_id)
    if balance.available_atomic < amount:
        raise InsufficientFunds()
    tx.debit_available(account_id, amount)
    tx.record_withdrawal_intent(request_id, account_id, amount)
```

**Why this is safer:** A transaction and row lock make the balance check and debit atomic and recoverable.

**Trade-offs:** Row locks and transactions can cause contention, deadlocks, and lower throughput on hot accounts; keep transactions short, use appropriate isolation, and retry conflicts safely.

### 9.3 MUST — Handle duplicate events, reorgs, finality, and auditable corrections

> **AGENTS.md requirement:** Handle duplicate/out-of-order events, reorgs, and chain-specific finality before treating settlement as final. Ledger postings must balance and corrections must remain auditable.

**Violation**

```python
# Credits immediately for each webhook without checking finality.
ledger.credit(event.user_id, event.amount_atomic)
```

**Risk:** Duplicate events or reorganized blocks can create unbacked credits and unauditable balances.

**Compliance**

```python
if not chain_policy.is_final(event.block_hash, event.block_height):
    store.record_pending_deposit(event)
else:
    with ledger.transaction() as tx:
        if tx.mark_event_processed_once(event.unique_event_id):
            tx.post_balanced_deposit(event)  # Equal debits and credits.
# Reorgs/corrections use new auditable reversal entries, never history edits.
```

**Why this is safer:** Finality checks, event deduplication, and reversal entries preserve an auditable ledger.

**Trade-offs:** Waiting for finality delays deposits, while deduplication and reversal entries add storage and processing work; confirmation thresholds must reflect each chain's risk and UX needs.

## 10. Verify continuously and report evidence

### 10.1 MUST — Run available formatting, type, lint, and test checks

> **AGENTS.md requirement:** Run the project's formatter/linter, type checker, and relevant tests when available; introduce or run strong static checks for financial paths. Do not knowingly leave new warnings or failing checks unexplained.

**Violation**

```text
"Looks correct to me; no need to run checks."
```

**Risk:** Skipping automated checks leaves preventable defects and regressions undetected.

**Compliance**

```bash
ruff format --check .
ruff check .
pyright
pytest -q
```

**Why this is safer:** Repeatable automated checks catch style, type, and behavior regressions before release.

**Trade-offs:** Linting, typing, and test suites consume developer/CI time and can slow iteration; incremental runs and parallel CI can reduce feedback time without skipping checks.

### 10.2 MUST — Test adversarial paths without real assets

> **AGENTS.md requirement:** Test safety-critical paths for boundary values, precision, invalid inputs, duplicate requests, races, timeouts after submit, crashes, failed dependencies, and chain reorgs where applicable. Never test with real keys or live funds.

**Violation**

```python
def test_transfer():
    assert transfer(1) == "submitted"  # Only the happy path.
```

**Risk:** Happy-path tests miss precisely the conditions that cause irreversible financial losses.

**Compliance**

```python
@pytest.mark.parametrize("amount", [0, -1, 1, MAX_AMOUNT + 1])
def test_withdrawal_boundaries(fake_ledger, amount):
    assert_withdrawal_policy(fake_ledger, amount)

# Additional isolated tests inject duplicate requests, concurrent debits,
# timeout-after-submit, restarts, dependency failures, and chain reorgs.
# Use fake signers, simulated chains, and synthetic keys/funds only.
```

**Why this is safer:** Boundary and failure-injection tests exercise costly failure modes without risking real funds.

**Trade-offs:** Fault-injection and property tests lengthen test suites and require mocks or simulators that may differ from production; balance fast unit tests with realistic integration tests.

### 10.3 MUST — Report checks actually run and remaining unknowns

> **AGENTS.md requirement:** Report which checks actually ran, what could not be verified, and any unresolved safety deviations. Never claim unrun tests passed.

**Violation**

```text
"All tests passed."  # No tests were executed.
```

**Risk:** False confidence about verification can cause untested financial code to ship.

**Compliance**

```text
Checks: ruff check . (pass), pytest tests/ledger (pass, 24 tests).
Not run: pyright (not installed); reorg integration suite (no local node).
Open risk: transaction confirmation behavior under deep reorgs is unverified.
```

**Why this is safer:** Accurate verification reports reveal exactly what was checked and what remains uncertain.

**Trade-offs:** Detailed reporting takes time and can become noisy if every minor check is listed; concise, accurate evidence makes review faster and avoids unsupported assurances.

---

## Agent workflow

The [`AGENTS.md`](./AGENTS.md) workflow is procedural guidance, **not an eleventh rule**: identify financial risk, state invariants and unknown outcomes, make the smallest safe change, run checks, and report unverified behavior or MUST conflicts.

## Attribution

Adapted from Gerard J. Holzmann, *The Power of Ten: Rules for Developing Safety-Critical Code* (2006). [Overview and original rule references](https://en.wikipedia.org/wiki/The_Power_of_10:_Rules_for_Developing_Safety-Critical_Code).
