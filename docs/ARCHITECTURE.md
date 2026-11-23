# Architecture

How Melamu Holdings OS is organised. The source code is private; this page describes its structure and the rules it follows.

## Overview

```
Browser (Angular 21)  ──HTTP / JSON───>  ASP.NET Core API (.NET 10)  ──EF Core──>  SQL Server
  standalone components, signals          controllers per module                   trading tables (dbo)
  lazy routes per module                  application services                     finance schema
  no zone.js                              domain rules, unit-tested                handbook schema
```

Every day the API runs on the owner's computer at `localhost` and serves the built web app itself, under its own Content Security Policy. A move to a hosted server is planned, not done.

## Backend layers

| Layer | Depends on | Holds |
|---|---|---|
| **Domain** | nothing | Entities and rules: `Money` and `Currency`, the ledger and its entries, funds, accounts, payouts, allocation policies and the allocation calculator, obligations, continuity, budgets, planned purchases, trades and journal templates, strategies and their rules, prop firm accounts and their rules, goals, the Handbook, and business time |
| **Application** | Domain | Use cases as services: ledger posting, payouts, allocation (preview, confirm, settle), finance read models, the overview, analytics, the journal, import with reconciliation, the optional AI review, goals, what-ifs, cash flow, the Handbook. Repository interfaces |
| **Infrastructure** | Application, Domain | EF Core `DbContext`, configurations per module, migrations and repositories; file storage for attachments outside the web root; the OpenAI-compatible AI client |
| **API** | all of the above | Controllers per module, authentication and authorisation, ProblemDetails errors, security headers, hosting the web app |

Modules depend in one direction. Finance knows nothing about Trading; the two meet only through explicit links (a payout can name the prop firm account it came from, and an expense can be linked as an account's fee). Planning reads Finance and Trading, and neither reads Planning.

## Rules for money

- **Money is a value with a currency.** Amounts are held in minor units, and adding two currencies is an error, not a conversion.
- **The ledger is append-only.** Every movement is a transaction of entries on an account and a fund. Income and expenses change the total; transfers and earmarks between funds or accounts net to zero, so moving money never creates or loses any. A mistake is corrected by a reversal, once, with a reason.
- **One posting path.** Every ledger transaction goes through a single service that checks the idempotency key and saves the transaction together with whatever depends on it (a payout receipt, an allocation settlement) and its audit event, in one database transaction.
- **Balances are never stored.** They are summed from entries, per account, fund and currency, optionally as of a date. The browser never computes an authoritative balance.
- **Concurrency.** Unique indexes stop duplicates (idempotency keys, one live allocation per payout), and row versions stop two requests changing the same payout or allocation at once; the loser gets HTTP 409.
- **Allocation.** A policy is a versioned set of percentages that must total 100%. Splitting rounds each line down to the cent and gives the leftover cents to the largest remainders, so the lines add up exactly. A confirmed allocation keeps the policy version, fund names and amounts it was made with.
- **Ownership is explicit.** Company and personal money are never mixed: personal accounts are shown apart and never counted as business cash.
- **Projections are never money.** Requested payouts, what-ifs and planned purchases never appear in a cash, income or balance figure.
- **"Today" follows the business time zone** (South African time by default, switchable to New York time), not the server's clock.

## Trading

- **One trade model.** A journal entry is the trade together with its template answers. Trades entered by hand and trades imported from CSV feed the same analytics, so the figures can never disagree.
- **Statuses:** Draft → Open → Closed → Reviewed. Analytics count only trades with complete execution data.
- **Versioned templates and rules.** Editing a journal template or a strategy's rules never reinterprets past entries.
- **Every change to a trade is recorded** with its values before and after.
- **Prop firm rules are data,** because firms differ in drawdown model, payout rules and evaluation structure.

## Frontend

- **Feature modules:** each has its own lazy routes and API service:
  - Overview;
  - Trading (journal, trades, performance, behaviour, strategies, templates, import, prop accounts);
  - Finance (payouts, allocations, accounts, funds, transactions, obligations, continuity, investments, budgets, purchases, audit log);
  - Business (goals, what-ifs, cash flow, allocation history, Handbook);
  - Settings.
- **Signals for all component state,** with zoneless change detection.
- **Loading, empty and error states with Retry** on every page. Forms that move money send an idempotency key that is kept for retries and renewed when the values change.
- **Its own design system:** tokens, IBM Plex type and shared components. The status words mean the same everywhere, and the layout works down to phone width.
- **Accessibility:** axe-core runs against every page in the test suite. There is a skip link, focus moves to each new page's heading, and wide tables are scrollable, focusable regions.

## Security

- **Sign-in is required by default** on every endpoint, with a test that walks all of them. Finance requires the Owner role, and trading data is scoped to its user.
- **Sign-in:**
  - JWT bearer tokens and BCrypt hashing.
  - Login rate limiting.
  - The same response time for known and unknown emails.
  - No public sign-up; the owner account is created from the command line.
- **Security headers on every response,** a strict Content Security Policy for the web app, and `default-src 'none'` for the API.
- **Uploads:**
  - **Attachments** are checked by their content and size, stored under random names outside the web root, and served only to their owner.
  - **CSV import** has row-level validation and runs in one transaction.
- **The optional AI review** sends journal text as data, with no tools. A reply is never saved without the trader.
- **Secrets:** the signing key comes from user secrets or environment variables and is required to start. No secret is in the repository.

## Data and operations

- **Schema changes go only through EF Core migrations.** Each was applied to the real database after a verified backup. A test builds a database from empty and rolls the latest migrations down and up again.
- **Backups:** scripts back up and restore the database and attachments with checksums. A restore goes into a copy by default.
- **A separate demo database** with fictional records produces the screenshots. It is marked as a demo, and the app labels every page.

## Testing

| Suite | What it covers |
|---|---|
| Unit (xUnit) | Money and currency, ledger rules, allocation and rounding, continuity, budgets, goals, business time, analytics definitions, the journal, prop firm rules |
| API integration (xUnit, real SQL Server, one database per test class) | Every key financial operation end to end through HTTP: recording, balances, allocation, settlement, reconciliation, summaries, validation, 401/403/404/409/422, concurrency and constraint violations |
| Angular (Vitest) | Pages and forms: loading, empty and error states, validation, double submission, idempotency keys, previews, routing, accessibility of every page |
