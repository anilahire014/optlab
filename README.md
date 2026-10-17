# optlab

A read-only analytics and paper-trading platform for Indian index options and NSE equities, built to answer one question honestly: **does a trading idea have an edge after costs?** Every hypothesis is pre-registered, tested against random entries, judged once, and recorded whatever the result.

Python · FastAPI · PostgreSQL · SQLAlchemy · React · TypeScript

> **The source code is private.** This repository documents the architecture and engineering.
> Happy to walk through the codebase or grant read access on request — **anilahire014@gmail.com**

---

## At a glance

| | |
|---|---|
| **Backend** | Python 3.13, FastAPI, SQLAlchemy 2, Alembic, pydantic-settings |
| **Database** | PostgreSQL 18 — 57 tables, 47 migrations, native partitioning |
| **Frontend** | React, Vite, TypeScript, TanStack Query, Lightweight Charts |
| **Real-time** | Server-Sent Events; inter-process messaging via `LISTEN/NOTIFY` |
| **Data** | Zerodha Kite Connect (WebSocket ticks + REST), NSE/BSE public files, XBRL filings |
| **Tests** | ~4,155 backend (pytest, integration against live PostgreSQL), ~150 frontend (Vitest) |
| **Quality** | ruff, mypy, tsc, 12 architecture decision records |

Solo project, in active development.

---

## What it does

**Market data pipeline.** A collector process streams live ticks over the Kite WebSocket and snapshots the full option chain every 5 seconds for NIFTY, BANK NIFTY and SENSEX. End-of-day bhavcopy files from both exchanges, cash-market data back to October 2018, delivery positions, FII/DII flows and quarterly XBRL filings are ingested on a nightly schedule.

**Options analytics.** Black-76 pricing off a forward derived from put-call parity, with implied volatility and Greeks computed per strike. Exact transaction costs from a dated rate table — brokerage, STT, exchange, IPFT, SEBI, stamp duty and GST — rounded the way a real contract note rounds, because costs decide most of the verdicts.

**Paper trading.** An order-matching engine with fills at the touch, all-or-nothing size checks, live stop and target monitoring on every chain update, and settlement at expiry. It cannot place a real order (see below).

**Research engine.** Walk-forward backtests, HAR-type volatility forecasting in shadow mode, and a setup lab for running rule sets against cached minute candles. Every idea is judged in two independent halves of the history and against random entries with identical instruments, days and exits.

**Web application.** Eighteen tabs across live market view, option chain, signals, paper account, trade journal, equity scanner, fundamentals and research — served by FastAPI from a built Vite bundle.

---

## Engineering worth pointing at

**Safety is enforced by the test suite, not by convention.** A test named `test_nothing_can_place_or_change_a_real_order` scans every source file and fails the build if any of them references a broker order call — `place_order`, `modify_order`, `cancel_order`, GTT or position conversion. The execution mode setting accepts exactly one value: `DISABLED`. Making the guarantee structural rather than a code-review promise means it cannot quietly erode.

**Layer boundaries are also a test.** The codebase is a modular monolith split into `domain` (pure logic, zero I/O), `ml`, `db`, `adapters`, `services`, `workers` and `api`. An architecture test asserts the import graph, so a service reaching into a router or domain logic importing a database session breaks CI. One consequence worth noting: the API layer must not load numpy at import time, which keeps request-path startup fast.

**Partitioning for data volume.** Raw ticks live in daily PostgreSQL partitions with a 7-day retention window, after which expired partitions are archived to gzipped CSV. Minute candles and chain snapshots are partitioned monthly. Per-strike, per-minute IV, Greeks and open-interest change are retained permanently — because Kite serves no intraday history for expired options, this table is the *only* record of how those contracts behaved.

**Real-time without a message broker.** Processes communicate through database tables and PostgreSQL `LISTEN/NOTIFY`; the browser receives updates over Server-Sent Events. One fewer moving part to operate, and it fit the deployment constraints.

**Credentials and auditability.** The broker access token is stored Fernet-encrypted at rest. The audit log is append-only, enforced by a database trigger that blocks updates and deletes.

**LLM integration with hard limits.** Gemini, Anthropic and OpenAI-compatible clients sit behind a single service that enforces a monthly rupee budget and a daily request cap, caches responses, and records every request. Only figures the application itself computed are sent; the paper account never leaves the machine.

**Built under real constraints.** The whole system runs on a 4 GB Windows laptop — native, no WSL, no Docker — with PostgreSQL as a Windows service and `just` recipes driving PowerShell. The partitioning strategy, the retention windows and the archive tiering exist because of that ceiling, not in spite of it.

---

## Research record

Thirty-seven ideas have been pre-registered and judged. **One passed.**

A post-earnings drift signal — buying at day two's close after a quarterly result that both surprised on earnings and was bought in the days after, where both measures sit in the top quintile, held twenty sessions — returned +3.66% net (t = 4.6) across 2019–22 and +2.73% (t = 3.0) across 2023–26. It is now being tracked forward on paper.

Everything else failed or is still running, including the project's own intraday signal rules. The ledger records all of them with their verdicts.

That ratio is the point. The discipline that produces it: pre-register the exact rules and the pass bar before running anything, compare against random entries, measure after real costs, never re-tune on data an idea has already seen, and apply Holm's correction when testing variants together. With roughly forty hypotheses at a 5% bar, two or three pass by luck alone — so a single pass is treated as weak evidence until it survives forward.

---

## Screenshots

<!-- Add 3-4 screenshots here. Suggested: the Live cockpit, the option chain with
     Greeks, the paper journal, and the research scoreboard. -->

---

## Documentation

The private repository carries architecture notes, 12 ADRs, per-module documentation, a runbook, a testing guide, the research playbook and the ideas ledger. Every change updates the README, CHANGELOG, the relevant module doc and the in-app help text.

---

**Not investment advice.** This is a personal research tool. It does not place real orders.
