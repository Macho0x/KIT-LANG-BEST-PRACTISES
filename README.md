# Kit Lang Trading Bot — Documentation Architecture (Kit Lang version 2026.4.24)

This project uses a **three-layer documentation system** designed for both human developers and AI agents. Each document has a distinct role, and together they ensure correct, safe, and maintainable code.

---

## `AGENTS.md` — The Compass (Always Loaded)

This is the **primary context document** for AI agents and the first file any developer should read. It contains the project's non-negotiable safety rules (e.g., never use `Type!` in trading code, never use `??` for market data), module architecture, naming conventions, and build commands. It also includes a **Quick FAQ** and a **Section Index** that let agents resolve syntax ambiguities by targeting specific line ranges in `SYNTAX.md` — without ever loading the full reference into memory.

---

## `SYNTAX.md` — The Map (On-Demand Reference)

A comprehensive language and style guide (~2,000 lines) that documents Kit Lang's syntax, type system, and idioms in depth. It is **not** loaded into an agent's context by default. Instead, `AGENTS.md` points to exact line ranges here when deeper detail is needed. Think of it as a reference manual you pull off the shelf for specific questions: *"What's the exact syntax for refinement type construction?"* or *"How do list patterns work?"*

---

## `GRAMMAR.md` — The Blueprint (Rarely Needed)

The formal EBNF grammar specification of Kit Lang. This is intended for parser-level questions, compiler contributions, or deep language-theory work. Both humans and agents should only consult this when `SYNTAX.md` is insufficient — which is rare for day-to-day trading bot development.

---

## How They Flow Together

| Step | Action | Document |
|------|--------|----------|
| 1 | Start here for every task | **`AGENTS.md`** |
| 2 | Need a syntax reminder? | Check **§9 FAQ** in `AGENTS.md` |
| 3 | Need deeper detail? | Use the **§9 Index** to fetch 20–60 lines from **`SYNTAX.md`** |
| 4 | Writing a parser or language tool? | Consult **`GRAMMAR.md`** |

---

## Golden Rule

> **`AGENTS.md`** tells you *what to do and why*.  
> **`SYNTAX.md`** tells you *how to write it*.  
> **`GRAMMAR.md`** tells you *how the language is formally defined*.

For trading bot development, you will spend 90% of your time in the first two, and almost none in the third.
