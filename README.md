# The Power of 10 for Python Crypto Systems

A small, auditable set of engineering rules for software handling cryptocurrency, keys, balances, transactions, and settlement.

Inspired by Gerard J. Holzmann's [*The Power of Ten: Rules for Developing Safety-Critical Code*](https://en.wikipedia.org/wiki/The_Power_of_10:_Rules_for_Developing_Safety-Critical_Code). This is an independent adaptation, not an official standard or a security guarantee. Like the original, it favors rules that make code easier to inspect, bound, and verify.

> **Simple enough to review. Bounded enough to operate safely. Precise enough to verify.**

## Use with Cursor or another coding agent

Copy [`AGENTS.md`](./AGENTS.md) into your repository root. **`AGENTS.md` is the normative ruleset**; this README is an illustrated companion, with one example pair for **every** MUST or SHOULD clause, in the same order.

- **MUST** — blocking requirement; the agent must not silently waive it.
- **SHOULD** — default practice; deviations need a concrete technical justification.

The examples are deliberately small and focus on one failure mode at a time. Helpers such as `ledger`, `store`, and `signer` are illustrative interfaces, not fully implemented production components; a compliant snippet demonstrates its named requirement, **not** a complete secure transaction workflow.
