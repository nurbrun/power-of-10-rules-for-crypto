# The Power of 10 for Crypto Systems

A small, auditable set of engineering rules for software handling cryptocurrency, keys, balances, transactions, and settlement.

Inspired by Gerard J. Holzmann's [*The Power of Ten: Rules for Developing Safety-Critical Code*](https://en.wikipedia.org/wiki/The_Power_of_10:_Rules_for_Developing_Safety-Critical_Code).

This is an independent adaptation, not an official standard or a security guarantee. Like the original, it favors rules that make code easier to inspect, bound, and verify.

> Simple enough to review. Bounded enough to operate safely. Precise enough to verify.

## Why this exists

In financial systems, seemingly minor software defects can have irreversible consequences. A rounding error can create incorrect balances, an unsafe retry can duplicate a withdrawal, and an improperly validated transaction can permanently transfer funds to the wrong destination.

The importance of rigorous coding standards extends well beyond cryptocurrency. Following Toyota's unintended acceleration recalls, a NASA investigation of its electronic throttle control software identified at least 243 violations of the original Power of 10 rules. The case demonstrates how numerous seemingly small departures from sound engineering practices can undermine confidence in safety-critical software.

The original Power of 10 demonstrated the value of distilling complex engineering practices into a small number of rules that developers can remember, reviewers can evaluate, and tooling can help enforce.

Today, humans and AI agents are increasingly sharing responsibility for writing software. AI can rapidly build functional prototypes by making reasonable assumptions, filling in missing requirements, and choosing implementations without constant human intervention. In non-critical systems, this can be an enormous productivity advantage. In financial systems, those same assumptions can introduce risks that are difficult to detect and expensive—or impossible—to reverse.

As we delegate more implementation decisions to AI agents, we are also delegating more trust. That trust needs clearly defined boundaries.

This project aims to translate established safety-critical engineering principles into explicit, actionable, and verifiable constraints for AI coding agents. Rather than relying on an agent to independently determine what constitutes acceptable risk, these rules establish expectations around correctness, financial precision, failure handling, and verification.

The goal is to make risk easier to identify, reason about, and control so that humans and AI agents can move fast without sacrificing safety.

## Available languages

| Language | Rules | Examples |
|---|---|---|
| Python | [AGENTS.md](python/AGENTS.md) | [README.md](python/README.md) |

## How to use

1. Choose your language from the table above.
2. Copy its `AGENTS.md` into your project's root directory.
3. Use a compatible AI coding agent, such as Cursor, to apply the rules when writing or modifying code.
4. Consult the language-specific README for examples, risks, benefits, and trade-offs.

The `AGENTS.md` file is the authoritative instruction set. The accompanying README explains the reasoning behind each requirement.

## Philosophy

Safety doesn't always mean more code, more abstractions, or more checks. Every additional layer introduces complexity that can itself become a source of defects.

The goal is to find the smallest set of constraints that meaningfully reduces risk while preserving performance, readability, and developer productivity.

**Correctness over cleverness. Explicit behavior over hidden assumptions. Evidence over confidence.**

## Attribution

Adapted from Gerard J. Holzmann's *The Power of Ten: Rules for Developing Safety-Critical Code* (2006).

This project is an independent adaptation for cryptocurrency software and is not affiliated with or endorsed by the original author.
