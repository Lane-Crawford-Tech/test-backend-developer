# Backend Code Test — Supplier Product Catalog (NestJS)

**Timebox: 2–4 hours.** Stop at 4 hours and ship what works — a small, complete implementation scores better than unfinished over-engineering.

---

## 1. Context

Suppliers send us product feeds as XML files. You will build a small **NestJS** service that:

- stores products in a database through an **ORM/ODM**,
- exposes **RESTful CRUD** endpoints,
- binds request data through **DTOs**,
- validates input with **class-validator** (global `ValidationPipe`),
- **imports products from an XML feed**, reporting per-row failures.

The database runs locally — either in the provided Postgres container or as a zero-setup SQLite file. A reviewer must be able to clone the repo, follow your `README.md`, and verify every behavior with `curl`.

### Repo contents provided to you

- `ASSIGNMENT.md` — this file (the assignment).
- `fixtures/sample-products.xml` — the feed used for the import scenario. Treat it as authoritative; do not modify it.
- `docker-compose.yml` — starts a local Postgres for development (see section 2).

Commit your finished service to this repo. **You must provide your own `README.md`** — see section 7. Keep or rename this file, but do not ship the repo without a `README.md`.

---

## 2. Tech requirements

| Concern | Requirement |
|---|---|
| Runtime | Node.js **>= 20 LTS** (pin the version you used, e.g. `.nvmrc`) |
| Framework | **NestJS** (latest stable) |
| Persistence | An ORM/ODM. **Postgres + TypeORM is recommended** — a `docker-compose.yml` is provided so the database starts with one command. **SQLite** (file-based, zero setup) is also acceptable. Prisma, MikroORM, or Mongoose + MongoDB are acceptable **only if** startup stays a documented one-command setup |
| DTO validation | `class-validator` + `class-transformer` |
| XML parsing | Any pure-JS parser (e.g. `fast-xml-parser`). Avoid native modules |
| Tests | Optional but encouraged: Jest (Nest default) + `supertest` for e2e |

No auth, no external queue/cache, no frontend. The service is a single Nest application listening on `PORT` (default `3000`), base path **`/products`** (no global prefix).

### Provided database container

The repo includes a `docker-compose.yml` that starts **Postgres 16** with these settings:

| Setting | Value |
|---|---|
| Host / port | `localhost:5432` |
| Database | `catalog` |
| User | `app` |
| Password | `app` |

Start it with `docker compose up -d` (the container healthcheck reports when the database is ready; `docker compose ps` shows the status). Data persists in the `pgdata` volume — run `docker compose down -v` to reset it. If port `5432` is already in use on the host, remap the host port in the compose file and adjust your connection settings accordingly.

---

## 3. Domain model

A **Product**:

| Field | Type | Rules |
|---|---|---|
| `id` | string (UUID) | server-generated |
| `sku` | string | **required, unique**, matches `^[A-Z0-9-]{3,32}$` |
| `name` | string | **required**, 1–200 chars |
| `description` | string | optional, max 2000 chars |
| `price` | number | **required**, `> 0`, at most **2 decimal places** |
| `currency` | string | optional, default `"USD"`, matches `^[A-Z]{3}$` |
| `stock` | integer | **required**, `>= 0` |
| `category` | string | **required**, 1–64 chars |
| `status` | `"active"` \| `"discontinued"` | optional, default `"active"` |
| `createdAt` / `updatedAt` | string | ISO 8601, server-managed |

How you persist `price` (integer cents, decimal column with transformer, etc.) is your choice. **The public API must always use a JSON number** (e.g. `49.99`), and read-back values must round-trip correctly.

---

## 4. API contract

All responses/timestamps are JSON. Every response body is JSON except `204 No Content`.

| Method | Path | Success | Possible errors |
|---|---|---|---|
| `POST` | `/products` | `201` + Product | `400`, `409` |
| `GET` | `/products` | `200` + List envelope | `400` |
| `GET` | `/products/:id` | `200` + Product | `404` |
| `PATCH` | `/products/:id` | `200` + Product | `400`, `404`, `409` |
| `DELETE` | `/products/:id` | `204`, empty body | `404` |
| `POST` | `/products/import` | `200` + Import summary | `400` |

### 4.1 `POST /products` — create

Request body (`CreateProductDto`): all fields above except `id`, `createdAt`, `updatedAt`. `currency`, `status`, `description` are optional; the rest are required.

```json
{
  "sku": "PRD-101",
  "name": "Bluetooth Speaker",
  "description": "Portable speaker with 12 h battery life.",
  "price": 49.99,
  "currency": "USD",
  "stock": 40,
  "category": "Electronics",
  "status": "active"
}
```

`201` response — the created product:

```json
{
  "id": "5f0c9f2e-8f3a-4c2b-9d61-2a1c7e4b9d10",
  "sku": "PRD-101",
  "name": "Bluetooth Speaker",
  "description": "Portable speaker with 12 h battery life.",
  "price": 49.99,
  "currency": "USD",
  "stock": 40,
  "category": "Electronics",
  "status": "active",
  "createdAt": "2026-11-19T10:15:00.000Z",
  "updatedAt": "2026-11-19T10:15:00.000Z"
}
```

- `400` — body fails validation, or contains unknown fields (see section 5).
- `409` — `sku` already exists.

### 4.2 `GET /products` — list with filtering and pagination

Query parameters:

| Param | Type | Default | Rules |
|---|---|---|---|
| `page` | integer | `1` | `>= 1` |
| `limit` | integer | `20` | `1 <= limit <= 100` |
| `q` | string | — | case-insensitive substring match against `sku` **or** `name` |
| `category` | string | — | exact match |
| `status` | string | — | **bonus**, exact match on `active`/`discontinued` |
| `minPrice` / `maxPrice` | number | — | **bonus**, applied to `price` |
| `sort` | string | — | **bonus**, e.g. `price:asc`, `createdAt:desc` |

Query values arrive as strings — convert them explicitly (e.g. `@Type(() => Number)`), don't rely on implicit coercion.

`200` response envelope (fixed contract):

```json
{
  "data": [ { "...product..." } ],
  "meta": { "page": 1, "limit": 20, "total": 42, "totalPages": 3 }
}
```

- `400` for invalid query values (e.g. `limit=500`, `page=0`, non-numeric `minPrice`).
- Any result ordering you use is fine, but if it matters, document it in the README.

### 4.3 `GET /products/:id`

`200` + Product, or `404` if no product has that id. (UUID only; no lookup by `sku`.)

### 4.4 `PATCH /products/:id` — partial update

Body: `UpdateProductDto` — every field optional, but **at least one property must be present**, otherwise `400`. Any supplied field must satisfy the same rules as create. `sku` may be changed, but it must remain unique (`409` on collision). Returns the updated product.

### 4.5 `DELETE /products/:id`

`204` with empty body, or `404`.

### 4.6 `POST /products/import` — import from XML

- `multipart/form-data` with a single file field named **`file`** (the XML feed). Max size **1 MB**.
- `400` if the file field is missing, the content is not well-formed XML, or the file contains **zero** `<product>` elements.
- Rows are processed **in document order**. Each `<product>` is validated with the same domain rules as the REST API:
  - valid row → **upsert by `sku`**: created if the SKU does not exist, updated if it does (this includes a SKU appearing earlier in the same file);
  - invalid row → skipped and reported; it does **not** fail the whole request.
- Re-importing an unchanged file is idempotent in content (no new rows).

`200` response — summary (contract is fixed; message strings are free-form):

```json
{
  "total": 11,
  "created": 7,
  "updated": 1,
  "failed": 3,
  "errors": [
    { "index": 8, "sku": "PRD-008", "message": "name is required" },
    { "index": 9, "sku": "PRD-009", "message": "price must be greater than 0" },
    { "index": 10, "sku": "PRD-010", "message": "stock must be an integer greater than or equal to 0" }
  ]
}
```

- `index` is the **1-based position** of the `<product>` element within the file.
- `sku` is the offending row's SKU, or `null` if it is missing/unreadable.
- `errors` is ordered by `index` ascending.
- If the same SKU appears twice in the file, the later row wins (it counts as an `updated`).

#### Sample feed (provided in `fixtures/sample-products.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<products>
  <product>
    <sku>PRD-001</sku>
    <name>Wireless Mouse</name>
    <description>2.4 GHz wireless mouse with USB receiver.</description>
    <price>19.99</price>
    <stock>120</stock>
    <category>Electronics</category>
  </product>
  <product>
    <sku>PRD-002</sku>
    <name>Mechanical Keyboard</name>
    <description>87-key tenkeyless keyboard, brown switches.</description>
    <price>89.50</price>
    <currency>USD</currency>
    <stock>45</stock>
    <category>Electronics</category>
  </product>
  <product>
    <sku>PRD-003</sku>
    <name>Noise Cancelling Headphones</name>
    <price>249.00</price>
    <currency>USD</currency>
    <stock>30</stock>
    <category>Electronics</category>
    <status>active</status>
  </product>
  <product>
    <sku>PRD-004</sku>
    <name>Standing Desk</name>
    <description>Height-adjustable desk, 120x70 cm.</description>
    <price>399.00</price>
    <currency>EUR</currency>
    <stock>12</stock>
    <category>Furniture</category>
  </product>
  <product>
    <sku>PRD-005</sku>
    <name>Ergonomic Chair</name>
    <price>289.00</price>
    <stock>18</stock>
    <category>Furniture</category>
  </product>
  <product>
    <sku>PRD-006</sku>
    <name>USB-C Hub</name>
    <description>7-in-1 hub with HDMI and card reader.</description>
    <price>59.99</price>
    <stock>75</stock>
    <category>Electronics</category>
  </product>
  <product>
    <sku>PRD-007</sku>
    <name>Desk Lamp</name>
    <price>34.90</price>
    <stock>60</stock>
    <category>Furniture</category>
    <status>discontinued</status>
  </product>
  <product>
    <sku>PRD-008</sku>
    <price>15.00</price>
    <stock>10</stock>
    <category>Accessories</category>
  </product>
  <product>
    <sku>PRD-009</sku>
    <name>Laptop Sleeve</name>
    <price>-5.00</price>
    <stock>25</stock>
    <category>Accessories</category>
  </product>
  <product>
    <sku>PRD-010</sku>
    <name>Monitor Stand</name>
    <price>45.00</price>
    <stock>abc</stock>
    <category>Furniture</category>
  </product>
  <product>
    <sku>PRD-002</sku>
    <name>Mechanical Keyboard</name>
    <description>87-key tenkeyless keyboard, brown switches - corrected price.</description>
    <price>84.00</price>
    <currency>USD</currency>
    <stock>45</stock>
    <category>Electronics</category>
  </product>
</products>
```

Expected results with this fixture on an **empty** database:

- First import: `total: 11`, `created: 7`, `updated: 1`, `failed: 3` (errors at indices 8, 9, 10). Note that `PRD-008` has no `name`, `PRD-009` has a negative price, and `PRD-010` has a non-numeric stock.
- Rows 1–7 create products. The final row repeats `PRD-002` with a corrected price (`84.00`), so it counts as one `update` — after import, `PRD-002` must have price `84.00` and description ending in "corrected price."
- Re-importing the same file: `created: 0`, `updated: 8`, `failed: 3`.

---

## 5. Validation and errors (DTO first)

- Request bodies and queries are bound to **DTO classes** — no bare interfaces or `any` for input.
- Register a **global `ValidationPipe`** with at least: `whitelist: true`, `forbidNonWhitelisted: true`, `transform: true`.
- **Unknown body fields → `400`.** Invalid `sku` format, bad price, wrong types, etc. → `400`.
- Validation also applies to query parameters on `GET /products` (with explicit type conversion).
- Error response shape: the NestJS default `BadRequestException` shape is fine, but each validation problem must be identifiable (property + constraint message). Example:

```json
{
  "statusCode": 400,
  "message": ["price must be a positive number", "stock must be an integer number"],
  "error": "Bad Request"
}
```

- The XML importer validates each row against the same domain rules. Reusing your DTO validators (e.g. `validate()` from `class-validator` on a plain object) is a plus, not a requirement.

Error status summary:

| Case | Status |
|---|---|
| Body/query validation failure, unknown fields | `400` |
| Malformed XML, missing file field, no `<product>` elements | `400` |
| Unknown id | `404` |
| Duplicate `sku` on create/patch | `409` |

---

## 6. Tests

We strongly encourage adding a few **e2e tests** (Jest + `supertest` is the Nest default) — they are a strong signal of engineering quality. Suggested coverage if you add them:

1. CRUD happy path: create → get by id → list → patch → delete → get returns `404`.
2. `400` on invalid create body (e.g. negative price) **and** `400` on an unknown field.
3. `400` on invalid list query (e.g. `limit=500`).
4. `404` on unknown id.
5. `409` on duplicate `sku` (create, and patch-into-collision).
6. Import: running the fixture file yields exactly `created: 7, updated: 1, failed: 3` with error indices `[8, 9, 10]`; a malformed-XML upload yields `400`.

If you include tests, use an **isolated database** (in-memory SQLite, temp file, or a separate test DB) so they never depend on or corrupt local dev data, and document the command in the README.

---

## 7. README.md requirements (mandatory)

A reviewer must be able to go from **clone → running → verified** without asking questions. Your `README.md` must include:

1. **Prerequisites** — exact Node.js version, package manager and version, Docker (if you use the provided Postgres), anything else needed. State explicitly how the database is started.
2. **How to start** — install command, configuration (env vars table with defaults, plus a committed `.env.example` if any env vars exist), dev command, production build + start command, expected port and a sample successful startup log line.
3. **Database** — how to start it (e.g. `docker compose up -d`), where data is stored, how to reset it, how to inspect it if useful.
4. **Project layout** — a directory tree with a one-line purpose for each meaningful folder/file, plus 2–3 sentences on the request flow (e.g. controller → service → repository).
5. **How to access / check the project** — endpoint table, request/response examples, and ready-to-paste `curl` commands for each endpoint, including the multipart import command.
6. **Validation and error format** — brief summary of what returns `400`/`404`/`409`.
7. **Tests (if included)** — command(s) to run and what is covered.
8. **Assumptions and trade-offs** — design decisions you made and their rationale.

Also commit a `.gitignore` that excludes `node_modules/`, build output, local DB files, and `.env`.

---

## 8. How this is evaluated

| Area | What we look for |
|---|---|
| Contract fidelity | Endpoints, statuses, and payload shapes match this document |
| Validation & DTO discipline | Typed DTOs, no `any` inputs, global pipe configured, query binding done properly |
| Import robustness | Well-formed parsing, per-row isolation of failures, correct upsert and counts |
| Error handling | Consistent, predictable statuses; no stack traces or 500s for client errors |
| Tests | Included tests are meaningful, isolated, and deterministic (a strong plus, not required) |
| README | Complete, accurate, runnable; explains layout and how to verify |
| Code quality | NestJS conventions, sensible layering, naming, no dead code |

Bonus points (optional, only if the core is done): `status`/price-range filters, sorting, Swagger/OpenAPI docs, mapping row-level validation errors back to DTO constraint names.

---

## 9. Submission

- Push the completed repository to a **private** GitHub repository, invite the account **`devsecops-lc`** as a collaborator, and send us an email to notify that the invitation has been sent.
- AI assistants are allowed. You own the code: expect to walk us through any part of it and extend it live.

Out of scope — do not implement: authentication, users/roles, frontend, file storage, caching, rate limiting, migrations tooling, CI/deployment, or containerizing the application itself (the provided `docker-compose.yml` only starts the database).
