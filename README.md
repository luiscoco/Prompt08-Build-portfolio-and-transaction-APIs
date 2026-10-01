# Build portfolio and transaction APIs

## Purpose of this learning activity

This prompt asks the coding agent to build authenticated portfolio CRUD endpoints and transaction APIs. **CRUD** means create, read, update, and delete; here, deleting a portfolio archives it instead of destroying its financial history. An **endpoint** is a URL that the frontend calls to read or change data.

Students learn how to turn financial rules into working backend behavior:

- Identify the user from a verified session and restrict access to that user's portfolios.
- Validate requests with shared Zod schemas before accepting financial data.
- Keep decimal quantities and money exact, including fractional shares and fees.
- Reject sales that would create a negative stock balance, including backdated trades.
- Prevent duplicate trades when a client retries a request.
- Make concurrent requests safe using PostgreSQL transactions and row locks.
- Test the actual Route Handlers against a real database.

These rules matter because a valid-looking request can still be unsafe: two simultaneous sales might each pass an independent balance check, and a repeated button click might otherwise record a purchase twice.

This activity implements the backend slice only. The React financial screens still show teaching fixtures; connecting those screens to these APIs is milestone 10. Valuation calculations are milestone 09.

## Steps actually performed

### 1. Read the project contract and existing implementation

The agent read `AGENTS.md`, [project state](docs/project-state.md), [milestone plan](docs/project-plan.md), architecture decisions, existing authentication, repositories, schemas, and tests. Existing session authentication from milestone 07 was reused rather than accepting a user ID in request JSON.

The installed Next.js 16.3.8 Route Handler documentation and Prisma 7.10.0 transaction types were checked. Current online Prisma documentation describes newer ORM 8 APIs, so the implementation kept the installed ORM 7 interface and did not upgrade dependencies.

### 2. Install the pinned workspace dependencies

```powershell
npm ci --no-audit --no-fund
```

The installation added 347 packages. On this Windows host the normal npm launcher refused delegated execution. The agent used the already installed npm CLI through Node instead:

```powershell
node 'C:/Users/luisc/AppData/Local/Author Software/nvm/installs/v24.21.0/node_modules/npm/bin/npm-cli.js' ci --no-audit --no-fund
```

That path is specific to the verification machine. Students with a working npm installation should use the normal commands in this guide.

### 3. Add shared request validation and database fields

`packages/contracts/src/index.ts` gained portfolio creation/edit schemas, transaction creation validation, pagination validation, an idempotency-key schema, and the `CONFLICT` error code.

`packages/db/prisma/schema.prisma` and the new migration at `packages/db/prisma/migrations/20261001080000_portfolio_api/migration.sql` added:

- `archivedAt` to preserve deleted portfolios and their transactions.
- `ledgerOrder`, a database-generated sequence used to break timestamp ties.
- `idempotencyKey` and `requestFingerprint`, with a unique key per portfolio.

A **migration** is a versioned SQL change to the database structure. This migration preserves the previous timestamp/ID order for existing trades before introducing sequence ordering.

### 4. Implement exact ledger rules and application services

New files `packages/domain/src/ledger.ts` and `ledger.test.ts` provide an exact inventory check. A **ledger** is the chronological list of recorded trades. The validator checks the balance after every entry, not just the final balance.

The new `packages/db/src/portfolio-service.ts` coordinates validation and database operations. It uses BigInt integers representing ten decimal places rather than JavaScript floating-point numbers for financial arithmetic. Transaction amounts round half up to ten decimal places, matching the database constraint.

Trades, renames, and archival acquire a PostgreSQL portfolio row lock inside a database transaction. A **row lock** makes a conflicting request wait until the current operation commits or rolls back. Recognized lock/deadlock/serialization conflicts get at most three attempts; exhausted contention returns HTTP 503. Redis is not required for this protection.

The service validates the full ledger after inserting the proposed trade inside the transaction. If any chronological balance becomes negative, the insertion rolls back.

### 5. Add thin authenticated Route Handlers

Created or extended:

- `apps/api/app/api/portfolios/route.ts`
- `apps/api/app/api/portfolios/[id]/route.ts`
- `apps/api/app/api/portfolios/[id]/transactions/route.ts`
- `apps/api/lib/portfolio-http.ts`
- `apps/api/lib/authorization.ts` and `http.ts`

The handlers authenticate first, call the application service, and return JSON. Business rules remain outside the handlers. Database repositories/exports, domain exports, the DB workspace manifest, and lockfile were also updated. The DB package now declares dependencies on the existing contracts/domain workspaces; external dependency versions did not change.

### 6. Test, correct failures, and document the policy

`apps/api/lib/portfolio.integration.test.ts` added eight PostgreSQL acceptance tests. An **integration test** checks collaborating parts together: these tests call the actual handlers with real signed session cookies and PostgreSQL storage.

The first run used AAPL, which is not in the synthetic seed. The fixture was corrected to ACME. A lock-timeout test then exposed Prisma adapter error codes nested under `originalCode`; retry classification was corrected and the test passed.

Fresh migrations ran on `portfolio_m08_verify`. The additive migration also ran on the existing disposable `portfolio_m07_auth_verify` database. The standalone milestone 06 database verifier was updated for archival, safe missing-resource errors, and BigInt serialization, but was not rerun.

The agent added [ADR 0004](docs/decisions/0004-transaction-correction-and-concurrency.md), [lesson 08](docs/lessons/08-portfolio-and-transaction-apis.md), and updated the decision index, versions notes, and project state. An **ADR** is an architecture decision record: it explains a design choice and its tradeoffs.

## Results achieved

### Available endpoints

All portfolio endpoints require a signed-in user. Requests for another user's portfolio return the same safe 404 response as a missing portfolio.

| Method | URL | Behavior |
| --- | --- | --- |
| GET | `/api/portfolios` | List owned portfolios, including archives |
| POST | `/api/portfolios` | Create a portfolio; returns 201 |
| GET | `/api/portfolios/:id` | Read an owned portfolio |
| PATCH | `/api/portfolios/:id` | Rename an active portfolio |
| DELETE | `/api/portfolios/:id` | Archive it; repeating the request is safe |
| GET | `/api/portfolios/:id/transactions?limit=50&offset=0` | Read a chronological page with `nextOffset` |
| POST | `/api/portfolios/:id/transactions` | Record an immutable trade; requires `Idempotency-Key` |

An **idempotency key** identifies one intended trade. Repeating the same request with the same key returns the original transaction and `replayed: true`. Reusing the key with a different trade returns 409. Using a different key intentionally creates a separate trade.

### Financial validation and edit policy

- Only existing USD stock identities are accepted, identified by symbol and exchange MIC. A **MIC** identifies a market; ACME/XNAS and ACME/XNYS have separate inventory.
- Quantity and price must be positive decimal strings; fees must be nonnegative strings. At most 18 integer digits and 10 fractional digits are allowed. Numbers, exponents, signs, and leading zeros are rejected.
- Dates must be valid UTC ISO strings with three fractional digits, such as `2025-01-02T00:00:00.000Z`. Future trades are rejected.
- Sell fees cannot exceed gross proceeds. Amounts outside database precision are rejected.
- Sales cannot exceed inventory at any point in the chronological ledger. Equal timestamps use database posting sequence order.
- Posted trades cannot be edited or deleted through the API. Corrections use explicit compensating BUY/SELL entries when they preserve valid inventory. They do not erase historical accounting effects.
- Portfolio edits change only the name. Archived portfolios remain readable, reject new trades/renames, and allow exact idempotency replays. There is no restore endpoint.

### Observed verification results

These results were observed during milestone implementation, not rerun merely to write this README:

| Check | Observed result |
| --- | --- |
| Full workspace tests with both test database URLs configured | 48 tests passed |
| API tests within that run | 25 passed, including eight new portfolio integration tests and eight existing authentication tests |
| Full typecheck and build | Passed |
| Final API rebuild after retry correction | Passed |
| Prisma schema validation | Passed |
| Fresh migration and upgrade of disposable authentication DB | Passed |
| Browser dependency boundary check | Passed |

Acceptance verified invalid input with zero writes, cross-user access, overselling, backdated trades, timestamp ties, exchange separation, duplicate/reused keys, concurrent conflicting sales, archive races, pagination, exact tiny fractions, rounding, and bounded lock retries followed by recovery.

## How to run locally

### Prerequisites

- Node.js 24.21.0 and npm 11.19.0, matching the pinned toolchain.
- Docker with a running engine and Compose for local PostgreSQL/Redis, or equivalent configured local services.
- Available ports 5173 (web), 3001 (API), 5432 (normal PostgreSQL), and 6379 (Redis).
- For the isolated acceptance setup below, port 5546 for a separate PostgreSQL container.

The demo needs no AI, market-data, or Microsoft Entra credentials. Demo authentication is explicitly local-only; production refuses it. Example database credentials below are public development values, not production secrets.

### Install, configure, migrate, and seed

Run these commands from the repository root in PowerShell:

```powershell
node --version
npm --version
npm ci --no-audit --no-fund
npm run infra:start

# Copy only if the file does not already exist. Otherwise merge the settings.
if (-not (Test-Path apps/api/.env.local)) {
  Copy-Item apps/api/.env.example apps/api/.env.local
}

$env:DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5432/portfolio_pilot'
$env:NODE_ENV='development'
$env:ALLOW_DEMO_SEED='true'
npm run build:types
npm run migrate:deploy --workspace @portfolio-pilot/db
npm run seed:demo --workspace @portfolio-pilot/db
npm run dev
```

In `apps/api/.env.local`, ensure `DEMO_AUTH_ENABLED=true`, `AUTH_BASE_URL=http://localhost:5173`, and the correct local database/Redis URLs. Keep Entra settings absent for this demo. Seeding explicitly creates fictional users and stock data; it is not a way to correct financial records.

Open exactly **http://localhost:5173**, then choose Alice. The frontend proxies `/api` to the API server, so the browser uses one origin. An **origin** is the protocol, host, and port together; mutation requests must match the configured origin. Do not substitute `127.0.0.1` in the browser URL.

The startup/browser sequence above is a demonstration procedure. The new milestone 08 routes were verified through direct handler integration tests, not a live browser or network smoke test.

## How to verify the API manually

After signing in, open the browser developer tools Console and run:

```javascript
const send = async (path, method = 'GET', body, key) => {
  const response = await fetch('/api/' + path, {
    method,
    headers: {
      'Content-Type': 'application/json',
      ...(key ? { 'Idempotency-Key': key } : {})
    },
    ...(body === undefined ? {} : { body: JSON.stringify(body) })
  });
  return { status: response.status, body: await response.json() };
};

const created = await send('portfolios', 'POST', {
  name: 'Lesson 08 ' + Date.now(), currency: 'USD'
});
const id = created.body.portfolio.id;
const trade = {
  security: { symbol: 'ACME', exchangeMic: 'XNAS', currency: 'USD' },
  side: 'BUY', quantity: '10', price: '100', fees: '2',
  occurredAt: '2025-01-02T00:00:00.000Z'
};

await send(`portfolios/${id}/transactions`, 'POST', trade, 'lesson-buy-1');
await send(`portfolios/${id}/transactions`, 'POST', trade, 'lesson-buy-1');
await send(`portfolios/${id}/transactions`, 'POST',
  { ...trade, side: 'SELL', quantity: '11' }, 'lesson-sale-1');
await send(`portfolios/${id}/transactions?limit=1&offset=0`);
await send(`portfolios/${id}`, 'PATCH', { name: 'Renamed ' + id });
await send(`portfolios/${id}`, 'DELETE');
await send(`portfolios/${id}/transactions`);
```

Expected behavior, also covered by the integration tests: the repeated purchase returns the same transaction ID with `replayed: true`; the quantity-11 sale returns 409; archival keeps the purchase history. The purchase amount is the string `"1002"` (10 × 100 + 2).

An illustrative oversell response has this shape; `requestId` is generated for the request:

```json
{
  "error": {
    "code": "CONFLICT",
    "message": "Trade would oversell the chronological ledger.",
    "requestId": "<generated UUID>"
  }
}
```

Save `id`, sign out, sign in as Bob, and request that ID. Expected status: 404. This browser exercise remains a manual verification step, not a claimed observed browser result.

## How to run automated checks

### Local checks without integration database URLs

```powershell
npm run typecheck
npm run build
npm run test
node scripts/check-browser-boundary.mjs
```

Without the test database URLs, the PostgreSQL integration suites explicitly skip. A successful unit-only run does not prove concurrency or database ownership behavior. Some workspaces currently have no tests and report that fact.

### PostgreSQL integration checks

Use disposable test databases, not the application's database. The tests require the exact database names below on localhost. They create unique test portfolios and clean up those portfolios; synthetic seed rows and test sessions remain in the disposable databases.

If the verification container from milestone 06 already exists, reuse it. If it is stopped, start it with `docker start portfolio-pilot-m06-verify`. If it does not exist, the setup documented in lesson 06 is:

```powershell
docker run --detach --name portfolio-pilot-m06-verify --publish 127.0.0.1:5546:5432 --env POSTGRES_USER=portfolio_local --env POSTGRES_PASSWORD=local_only_change_me --env POSTGRES_DB=portfolio_m06_verify postgres:17.6-alpine
docker exec portfolio-pilot-m06-verify pg_isready -U portfolio_local -d portfolio_m06_verify
```

Wait for `pg_isready` to report that PostgreSQL is accepting connections. Create each database only once; if it already exists, skip its creation command:

```powershell
docker exec portfolio-pilot-m06-verify psql -U portfolio_local -d postgres -c 'CREATE DATABASE portfolio_m08_verify'
docker exec portfolio-pilot-m06-verify psql -U portfolio_local -d postgres -c 'CREATE DATABASE portfolio_m07_auth_verify'

$env:DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5546/portfolio_m08_verify'
npm run migrate:deploy --workspace @portfolio-pilot/db
$env:PORTFOLIO_TEST_DATABASE_URL=$env:DATABASE_URL

$env:DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5546/portfolio_m07_auth_verify'
npm run migrate:deploy --workspace @portfolio-pilot/db
$env:AUTH_TEST_DATABASE_URL=$env:DATABASE_URL

npm run test --workspace @portfolio-pilot/api
npm run test
```

The observed API result with both URLs was:

```text
Test Files  5 passed (5)
     Tests  25 passed (25)
```

The complete workspace run passed 48 tests. Redis is not required for these portfolio integration checks. Restore your application `DATABASE_URL` before returning to development; using a fresh terminal avoids carrying test environment variables into the app.

## Limitations, failures, and unfinished work

- **Corrected failures:** the initial AAPL test fixture and Prisma lock-timeout error classification failed tests before being fixed. Final acceptance passed.
- **Environment obstacles:** npm's launcher trust restriction and Prisma cache `EPERM` initially blocked commands. The installed npm CLI and permitted engine-cache access allowed verification to finish. Docker inspection also needed access to the local engine. Students need equivalent permissions; no final milestone check remains blocked by these issues.
- **Warnings:** existing Vite `use client` directive warnings and three Next.js instrumentation Node/Edge analysis warnings remain. Builds passed despite them.
- **Skipped/unverified:** integration tests skip without their URLs. The historical standalone milestone 06 verifier was not rerun. No live browser/network smoke test was performed for the new endpoints. Real Microsoft Entra sign-in remains unverified without tenant credentials.
- **Pagination:** offset pages are not a snapshot across requests. Backdated inserts can shift rows; restart from offset zero after changes.
- **Scale:** full ledger validation scans the portfolio's entire history. This favors correctness for the teaching slice but needs evaluation for very large histories.
- **Corrections:** compensating entries are additional financial records, not silent reversals of historical profit/cost calculations. Corrections that cannot preserve long-only inventory need a future audited workflow.
- **Future milestones:** valuation, financial UI integration, archive restoration, and broader correction tooling are not implemented here. There are no trade edit/delete endpoints or broker execution.
- **Deployment:** no public deployment or paid cloud resources were created. This folder was not a Git repository; no commit or push was performed.

## Further reading

- [Project contract](AGENTS.md)
- [Current project state and actual check results](docs/project-state.md)
- [Milestone plan](docs/project-plan.md)
- [Pinned versions](docs/versions.md)
- [Lesson 07: authentication and authorization](docs/lessons/07-authentication-and-authorization.md)
- [Lesson 08: portfolio and transaction APIs](docs/lessons/08-portfolio-and-transaction-apis.md)
- [ADR 0004: correction and concurrency policy](docs/decisions/0004-transaction-correction-and-concurrency.md)
