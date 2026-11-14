# IAM Microservice

A study project: a small authentication service built with **Fastify, TypeScript, Prisma and PostgreSQL**, organized in a Clean-Architecture-inspired layout (routes → controllers → use cases → repositories).

> **Status:** prototype / learning project. It covers registration, login, a protected profile route and an access/refresh token flow. It is **not production-ready** — see [Known limitations](#known-limitations) for what is missing or incomplete.

I wrote about building it (in Portuguese) on my blog: [O que aprendi construindo um Microsserviço IAM do zero](https://gabrielmramos.dev/blog/construindo-microservico-iam-do-zero/).

---

## Endpoints

| Method | Route | Auth | Description |
|---|---|---|---|
| `POST` | `/users` | – | Register a user (`name` ≥ 3 chars, valid `email`, `password` ≥ 6 chars). Returns `201`, or `409` if the email already exists. |
| `POST` | `/sessions` | – | Log in with `email` + `password`. Returns `{ "token": "<access token>" }` and sets the `refreshToken` cookie. `401` on invalid credentials. |
| `GET` | `/me` | Bearer token | Returns the authenticated user's profile. |
| `PATCH` | `/token/refresh` | `refreshToken` cookie | Issues a new access token and a new refresh token cookie. `401` if the cookie is missing or invalid. |
| `GET` | `/docs` | – | Swagger UI (OpenAPI generated from the Zod schemas). |

Validation errors return `400` with the Zod error tree; domain errors (`AppError` subclasses) map to their own status codes; anything else returns `500`.

---

## How the auth flow works

```mermaid
graph TD
    A[Client] -- "email + password" --> B[POST /sessions]
    B -- "bcrypt compare" --> C{Valid?}
    C -- "No" --> C1[401 Invalid credentials]
    C -- "Yes" --> D["Access token (JWT, 10 min, carries role)"]
    C -- "Yes" --> E["Refresh token (JWT, 7 days)"]
    D -- "JSON { token }" --> A
    E -- "httpOnly cookie" --> A

    A -- "Access token expired" --> F[PATCH /token/refresh]
    F -- "Verify cookie signature and expiry" --> G{Valid?}
    G -- "No" --> G1[401 Invalid refresh token]
    G -- "Yes" --> H["New access token + new refresh cookie"]
    H --> A
```

- Both tokens are signed JWTs (`@fastify/jwt`, HS256 with `JWT_SECRET`).
- The refresh token is sent as a cookie with `httpOnly`, `secure` and `sameSite` (strict).
- The server keeps **no session state**: refresh tokens are not stored anywhere, so the refresh step only checks the token's signature and expiry.

---

## Tech stack

- **Runtime / language:** Node.js (ESM), TypeScript
- **HTTP:** Fastify 5 with `fastify-type-provider-zod`
- **Validation:** Zod 4
- **Auth:** `@fastify/jwt`, `@fastify/cookie`, `bcrypt` (cost factor 10)
- **Database:** PostgreSQL via Prisma 7 (`@prisma/adapter-pg`)
- **API docs:** `@fastify/swagger` + `@fastify/swagger-ui`
- **Tests:** Vitest, Supertest
- **Dev tooling:** `tsx`

---

## Project structure

```
src/
├── core/errors/                 # AppError + domain errors (with HTTP status codes)
├── env/                         # environment validation (Zod)
├── infra/
│   ├── database/prisma.ts       # Prisma client
│   └── http/
│       ├── app.ts / server.ts   # Fastify setup, plugins, error handler
│       ├── routes/              # route registration
│       ├── controllers/users/   # HTTP <-> use case glue
│       ├── middlewares/         # verifyJWT, verifyUserRole
│       └── schemas/             # Zod schemas (validation + OpenAPI)
└── modules/users/
    ├── use-cases/               # create-user, authenticate-user, get-user-profile (+ factories)
    ├── repositories/            # IUsersRepository + in-memory implementation (for tests)
    └── prisma/                  # Prisma implementation of the repository
prisma/                          # schema.prisma + migrations (users table, Role enum)
```

### How far the Clean Architecture goes

What holds:

- Use cases depend on an `IUsersRepository` interface, not on Prisma directly, and are built by factories.
- There is an in-memory repository, so use cases are unit-tested without a database.
- Routes, controllers and use cases are separate layers.

What does not (yet):

- The repository interface and the DTOs import types from `@prisma/client`.
- `create-user.use-case.ts` imports a type from the HTTP schema file (`infra/http/schemas`).
- There are no domain entities; use cases work with Prisma's `User` type.
- Controllers read the body with `request.body as any`.

---

## What is implemented

- User registration with email uniqueness check and bcrypt password hashing.
- Login with access token (10 min) and refresh token cookie (7 days).
- Protected `GET /me` route (`verifyJWT`).
- Refresh route that re-issues both tokens.
- `Role` enum (`ADMIN` / `MEMBER`, default `MEMBER`) stored in the database and included in the access token; a `verifyUserRole` middleware exists.
- Request validation with Zod and centralized error handling.
- Swagger UI generated from the schemas.
- Tests: 2 use-case spec files (create user, authenticate) with an in-memory repository, and 1 E2E spec (register → login → `GET /me`).

---

## Known limitations

**Security**

- **Refresh tokens are re-issued, not truly rotated.** Each refresh returns a new token, but the old one is never invalidated and there is no reuse detection (tokens are not stored). Real rotation is on the roadmap.
- **No revocation and no logout route.** There is no token store or blacklist; a stolen refresh token stays valid for up to 7 days.
- **Refresh does not check the user.** It verifies only the token; it does not confirm the user still exists.
- **Token types are not separated.** Access and refresh tokens use the same secret and have no distinguishing claim, so a refresh token (7 days) is accepted as a Bearer token on `GET /me`.
- **The role is lost after the first refresh.** The refresh token created at login has no `role` claim, so tokens issued by `/token/refresh` carry no role.
- **RBAC is scaffolding only.** `verifyUserRole` is not used by any route, and there is no way to create an `ADMIN` (registration always creates a `MEMBER`).
- **Hardcoded secrets.** `src/infra/http/app.ts` contains a hardcoded cookie secret, and `docker-compose.yml` contains a hardcoded `JWT_SECRET` and database credentials. Use environment variables and rotate anything that was committed.
- **No rate limiting** or brute-force protection on `/sessions`.
- **Basic password and email handling.** Minimum password length is 6; emails are not normalized (case-sensitive uniqueness); login for an unknown email returns before running bcrypt, so response timing can reveal whether an email exists.
- **Symmetric signing key.** Any service that needs to verify these tokens must share `JWT_SECRET`.

**Tests**

- Test coverage is small: one E2E happy path. The refresh flow, RBAC, error responses and `get-user-profile` are not covered.
- Several unit assertions only check `toBeInstanceOf(Error)`, and `create-user.spec.ts` contains a duplicated test.
- The E2E spec uses the real database configured in `.env` / `.env.test`, with no cleanup between runs. `npm test` also runs it (the Vitest pattern includes `*.e2e-spec.ts`), so point `.env` at a throwaway database.

**Project setup**

- **The `api` service in `docker-compose.yml` does not work as is:** its `DATABASE_URL` credentials (`docker:docker`) do not match the database service (`postgres` / `password123`), and the container runs `npm run dev`, which expects a `.env` file that is not in the repository. Use Docker only for the database for now (see below).
- The `Dockerfile` runs the dev server (`tsx watch`) and does not run migrations; there is no production build or `start` script.
- `package.json` metadata is incomplete (`license` says ISC while the `LICENSE` file is MIT, empty description/author, wrong `main`).
- The database has only a `users` table. No refresh-token or audit tables.

---

## Running locally

### Prerequisites

- Node.js 20.19+ (required by Prisma 7; the Dockerfile uses `node:20`)
- Docker and Docker Compose (for PostgreSQL)

### Setup

```bash
git clone https://github.com/gabrielmramos-dev/iam-microservice-node.git
cd iam-microservice-node
npm install

# Start only the database from docker-compose.yml
docker compose up -d database
```

Create a `.env` file in the project root whose `DATABASE_URL` matches the `database` service in `docker-compose.yml`:

```env
PORT=3333
NODE_ENV=dev
JWT_SECRET="change-me"
DATABASE_URL="postgresql://postgres:password123@localhost:5432/iam_db?schema=public"
```

Then apply the migrations and start the server:

```bash
npx prisma migrate dev
npm run dev
```

The API runs at <http://localhost:3333> and the docs at <http://localhost:3333/docs>.

### Try it

```bash
# register
curl -X POST http://localhost:3333/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Test User","email":"test@example.com","password":"password123"}'

# log in (-c saves the refresh cookie; the response contains the access token)
curl -i -c cookies.txt -X POST http://localhost:3333/sessions \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# profile (replace <token> with the access token from the login response)
curl http://localhost:3333/me -H "Authorization: Bearer <token>"
```

The refresh cookie is set with `secure`, so `curl` over plain `http://localhost` may not send it back; test `/token/refresh` from the Swagger UI or a client that treats localhost as a secure context.

### Tests

```bash
npm test            # Vitest in watch mode: unit specs + the E2E spec (needs a database)
npm run test:e2e    # E2E spec only, using .env.test
```

### Environment variables

| Variable | Default | Notes |
|---|---|---|
| `JWT_SECRET` | – (required) | Signing key for both tokens |
| `DATABASE_URL` | – (required by Prisma) | PostgreSQL connection string |
| `PORT` | `3333` | HTTP port |
| `NODE_ENV` | `dev` | `dev`, `test` or `production` |

---

## Roadmap

1. Fix the refresh flow: keep the `role` claim, separate access/refresh token types, check that the user still exists.
2. Store refresh tokens (hashed) to support real rotation with reuse detection, logout and revocation.
3. Rate limiting on authentication routes; email normalization; constant-time login path.
4. Use `verifyUserRole` on real routes and add an admin-only operation.
5. Move secrets out of the code and `docker-compose.yml`; fix the `api` compose service; add a production build.
6. Broaden tests: refresh, RBAC, error handler, E2E with an isolated, cleaned-up database.
7. Remove the Prisma and HTTP-schema imports from the repository and use-case layers.

---

## License

MIT — see [LICENSE](./LICENSE).
