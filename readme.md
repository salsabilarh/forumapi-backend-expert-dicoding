# 🗣️ Forum API — Production-Grade Discussion Platform

![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![Hapi.js](https://img.shields.io/badge/Hapi.js-FF6600?style=for-the-badge)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![Jest](https://img.shields.io/badge/Jest-100%25_Coverage-C21325?style=for-the-badge&logo=jest&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![NGINX](https://img.shields.io/badge/NGINX-Rate_Limited-009639?style=for-the-badge&logo=nginx&logoColor=white)
![CI/CD](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)
![Dicoding](https://img.shields.io/badge/Dicoding-Submission-4285F4?style=for-the-badge)
![Score](https://img.shields.io/badge/Score-5%2F5_%E2%98%85-FFD700?style=for-the-badge)
![Learning Path](https://img.shields.io/badge/Backend_JavaScript-Learning_Path_Stage_6-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)

> ⭐ **Final submission for "Menjadi Back-End Developer Expert dengan JavaScript"** — Stage 7 and capstone of the *Backend JavaScript Learning Path* at Dicoding.
> A production-structured RESTful discussion forum API implementing Clean Architecture across four strict layers, JWT dual-token authentication, RBAC, PostgreSQL with versioned migrations, 100% test coverage across 53 test files, automated CI/CD, and Docker + NGINX deployment. **Perfect score 5/5.**

---

## 📖 Table of Contents

- [Executive Summary](#executive-summary)
- [Key Features](#key-features)
- [Architecture Overview](#architecture-overview)
- [Tech Stack](#tech-stack)
- [Database Schema](#database-schema)
- [API Endpoints](#api-endpoints)
- [Request & Response Reference](#request--response-reference)
- [Installation & Local Usage](#installation--local-usage)
- [Testing](#testing)
- [Deployment](#deployment)
- [Performance Highlights](#performance-highlights)
- [What I Learned](#what-i-learned)
- [License](#license)

---

## Executive Summary

Forum API is the capstone of the Backend JavaScript Learning Path — the point where every concept from the preceding sixth stages converges into a single, coherent system: a discussion forum where users can create threads, post and delete comments, add and delete threaded replies, and toggle likes on comments.

What makes this project architecturally significant is not the feature set — it is the **discipline of separation**. Every use case is independently testable without touching a database, a framework, or an HTTP server. Every infrastructure concern (PostgreSQL, bcrypt, JWT) is injected as a dependency rather than imported directly. The result is a codebase where swapping the database or the web framework requires changes in exactly one place.

---

## Key Features

- **Clean Architecture (4 layers)** — Domains, Applications, Interfaces, and Infrastructures are strictly separated; use cases have zero knowledge of HTTP or PostgreSQL
- **JWT Dual-Token Authentication** — access token (short-lived) + refresh token (DB-backed, revocable) with `@hapi/jwt` strategy registration
- **Role-Based Authorization (RBAC)** — comment and reply deletion enforces owner-only access; attempting to delete another user's content returns `403 Forbidden`
- **Soft Delete Pattern** — deleted comments and replies are not removed from the database; their `is_delete` flag is set and their content is replaced with a placeholder at the use case layer, preserving thread readability
- **Batched N+1 Query Elimination** — like counts for all comments in a thread are fetched in a single `IN (...) GROUP BY` query, not per-comment round trips
- **Idempotent Like Toggle** — `PUT /threads/{threadId}/comments/{commentId}/likes` adds a like if none exists and removes it if it does, driven by a `SELECT` before the write decision
- **100% Test Coverage** — 53 test files across unit, integration, and functional test layers; enforced via Jest with `--coverage` flag
- **Automated CI/CD** — GitHub Actions pipeline builds, migrates, tests, and deploys on every push to master
- **Docker + NGINX Deployment** — single Dockerfile bundles Node.js app + NGINX reverse proxy; NGINX handles rate limiting (90 req/min on `/threads`) and request forwarding to the internal app port
- **DI Container** — `instances-container` manages all dependency injection, decoupling use case instantiation from infrastructure binding
- **Centralized Error Handling** — `onPreResponse` lifecycle hook translates domain errors to consistent HTTP responses; `DomainErrorTranslator` maps internal error codes to user-facing messages

---

## Architecture Overview

```
src/
├── Domains/                   ← Layer 1: Business entities & repository interfaces
│   ├── users/                    UserRepository (abstract), RegisterUser, RegisteredUser, UserLogin
│   ├── threads/                  ThreadRepository (abstract), NewThread, AddedThread, DetailThread
│   ├── comments/                 CommentRepository (abstract), NewComment, AddedComment
│   ├── replies/                  ReplyRepository (abstract), NewReply, AddedReply
│   ├── likes/                    CommentLikeRepository (abstract), CommentLike
│   └── authentications/          AuthenticationRepository (abstract), NewAuth
│
├── Applications/              ← Layer 2: Use cases & security interfaces
│   ├── use_case/                 11 use cases — each orchestrates Domains only
│   │   ├── AddThreadUseCase
│   │   ├── GetThreadDetailUseCase    (aggregates thread + comments + replies + likes)
│   │   ├── AddCommentUseCase
│   │   ├── DeleteCommentUseCase      (verifies ownership before delete)
│   │   ├── AddReplyUseCase
│   │   ├── DeleteReplyUseCase        (verifies ownership before delete)
│   │   ├── ToggleCommentLikeUseCase  (read → conditional add/delete)
│   │   ├── AddUserUseCase
│   │   ├── LoginUserUseCase
│   │   ├── LogoutUserUseCase
│   │   └── RefreshAuthenticationUseCase
│   └── security/                 AuthenticationTokenManager, PasswordHash (interfaces)
│
├── Interfaces/                ← Layer 3: HTTP delivery (Hapi plugins)
│   └── http/api/
│       ├── users/                handler + routes + plugin registration
│       ├── authentications/
│       ├── threads/
│       ├── comments/
│       ├── replies/
│       └── likes/
│
└── Infrastructures/           ← Layer 4: Concrete implementations
    ├── container.js              DI container — wires all abstractions to implementations
    ├── http/createServer.js      Hapi server — JWT strategy, plugin registration, error lifecycle
    ├── database/postgres/pool.js pg.Pool singleton
    ├── repository/               6 PostgreSQL repository implementations
    └── security/                 BcryptPasswordHash, JwtTokenManager
```

### Dependency Rule

Every dependency arrow points **inward only**:

```
Infrastructures → Interfaces → Applications → Domains
```

`Domains` has no imports from any other layer. `Applications` imports only from `Domains`. `Infrastructures` imports from all layers but is itself imported by nothing except `container.js`. This is the Clean Architecture dependency rule enforced as a structural constraint.

---

## Tech Stack

```
Runtime         Node.js 18 (Alpine in Docker)
Framework       @hapi/hapi v20 + @hapi/jwt
Database        PostgreSQL — 6 tables, versioned migrations via node-pg-migrate
Auth            JWT dual-token (access + refresh) + bcrypt password hashing
DI Container    instances-container
ID Generation   nanoid(16) — prefixed per resource type (thread-, comment-, reply-, like-)
Testing         Jest v27 — unit, integration, functional; 100% coverage
Linting         ESLint + eslint-config-airbnb-base
Reverse Proxy   NGINX — rate limiting, request forwarding
Containerization Docker (node:18-alpine + NGINX in single image)
CI/CD           GitHub Actions — build → migrate → test → deploy
```

---

## Database Schema

Six tables with enforced referential integrity via foreign keys and `CASCADE` delete:

```sql
users
  id VARCHAR(50) PK
  username VARCHAR(50) UNIQUE NOT NULL
  password TEXT NOT NULL
  fullname VARCHAR(50) NOT NULL

authentications
  token TEXT NOT NULL

threads
  id VARCHAR(50) PK
  title TEXT NOT NULL
  body TEXT NOT NULL
  owner VARCHAR(50) FK → users.id
  created_at TIMESTAMP

comments
  id VARCHAR(50) PK
  thread_id VARCHAR(50) FK → threads.id CASCADE
  owner VARCHAR(50) FK → users.id
  content TEXT NOT NULL
  is_delete BOOLEAN DEFAULT false    ← soft delete flag
  created_at TIMESTAMP

replies
  id VARCHAR(50) PK
  comment_id VARCHAR(50) FK → comments.id CASCADE
  owner VARCHAR(50) FK → users.id
  content TEXT NOT NULL
  is_delete BOOLEAN DEFAULT false    ← soft delete flag
  created_at TIMESTAMP

comment_likes
  id VARCHAR(50) PK
  comment_id VARCHAR(50) FK → comments.id CASCADE  INDEXED
  user_id VARCHAR(50) FK → users.id CASCADE         INDEXED
  UNIQUE(comment_id, user_id)                        ← prevents duplicate likes at DB level
```

---

## API Endpoints

### Authentication

| Method | Endpoint | Auth | Description |
|--------|----------|:----:|-------------|
| `POST` | `/users` | — | Register a new user |
| `POST` | `/authentications` | — | Login — returns access + refresh token |
| `PUT` | `/authentications` | — | Refresh access token |
| `DELETE` | `/authentications` | — | Logout — revokes refresh token |

### Threads

| Method | Endpoint | Auth | Description |
|--------|----------|:----:|-------------|
| `POST` | `/threads` | ✅ | Create a new thread |
| `GET` | `/threads/{threadId}` | — | Get full thread detail (comments, replies, like counts) |

### Comments

| Method | Endpoint | Auth | Description |
|--------|----------|:----:|-------------|
| `POST` | `/threads/{threadId}/comments` | ✅ | Add a comment to a thread |
| `DELETE` | `/threads/{threadId}/comments/{commentId}` | ✅ Owner | Soft-delete a comment |

### Replies

| Method | Endpoint | Auth | Description |
|--------|----------|:----:|-------------|
| `POST` | `/threads/{threadId}/comments/{commentId}/replies` | ✅ | Add a reply to a comment |
| `DELETE` | `/threads/{threadId}/comments/{commentId}/replies/{replyId}` | ✅ Owner | Soft-delete a reply |

### Likes

| Method | Endpoint | Auth | Description |
|--------|----------|:----:|-------------|
| `PUT` | `/threads/{threadId}/comments/{commentId}/likes` | ✅ | Toggle like on a comment (add if not liked, remove if liked) |

---

## Request & Response Reference

### Register User — `POST /users`

```json
// Request
{ "username": "johndoe", "password": "secret", "fullname": "John Doe" }

// Success 201
{
  "status": "success",
  "data": {
    "addedUser": { "id": "user-Qbax5Oy7L8WKf74l", "username": "johndoe", "fullname": "John Doe" }
  }
}
```

### Login — `POST /authentications`

```json
// Request
{ "username": "johndoe", "password": "secret" }

// Success 201
{
  "status": "success",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

### Get Thread Detail — `GET /threads/{threadId}`

```json
// Success 200
{
  "status": "success",
  "data": {
    "thread": {
      "id": "thread-h_W1Plfpj0TY7wyT2",
      "title": "Clean Architecture in Practice",
      "body": "A discussion on applying Clean Architecture...",
      "date": "2026-01-17T10:00:00.000Z",
      "username": "johndoe",
      "comments": [
        {
          "id": "comment-_pby2_tmXV6bcvcdev8xk",
          "username": "janedoe",
          "date": "2026-01-17T11:00:00.000Z",
          "content": "Great topic!",
          "likeCount": 3,
          "replies": [
            {
              "id": "reply-BErOXUSefjwWGW1Z10Ihk",
              "username": "johndoe",
              "date": "2026-01-17T12:00:00.000Z",
              "content": "Thanks for reading!"
            }
          ]
        },
        {
          "id": "comment-abc123",
          "username": "deleteduser",
          "date": "2026-01-17T13:00:00.000Z",
          "content": "**komentar telah dihapus**",
          "likeCount": 0,
          "replies": []
        }
      ]
    }
  }
}
```

---

## Installation & Local Usage

**Prerequisites:** Node.js v18+, PostgreSQL running locally

```bash
# Clone the repository
git clone https://github.com/salsabilarh/forum-api-dicoding.git
cd forum-api-dicoding

# Install dependencies
npm install

# Configure environment variables
cp .env.example .env
# Edit .env:
#   PGHOST=localhost
#   PGPORT=5432
#   PGDATABASE=forumapi
#   PGUSER=postgres
#   PGPASSWORD=yourpassword
#   ACCESS_TOKEN_KEY=your_access_token_secret
#   REFRESH_TOKEN_KEY=your_refresh_token_secret
#   ACCESS_TOKEN_AGE=3000

# Run database migrations
npm run migrate

# Start in development mode
npm run start:dev

# Start in production mode
npm start
```

Server starts at: **`http://localhost:3000`**

---

## Testing

The test suite covers three levels, all colocated with source files in `_test/` directories:

```bash
# Run all tests with coverage
npm test

# Watch mode (reruns on file change)
npm run test:watch

# Watch with coverage report
npm run test:watch:change
```

**Test structure:**

| Layer | Type | What is tested |
|---|---|---|
| `Domains/` | Unit | Entity constructors — field validation, type checking |
| `Applications/use_case/` | Unit | Use case orchestration — mock repositories injected |
| `Infrastructures/repository/` | Integration | SQL queries against a real test database |
| `Infrastructures/http/_test/` | Functional | Full HTTP request/response cycles via Hapi's `inject()` |

**Coverage target:** 100% across all instrumented files.

```bash
# Migrate the test database before running integration/functional tests
npm run migrate:test
```

---

## Deployment

### Docker

```bash
# Build image (bundles Node.js + NGINX)
npm run docker:build

# Run container (exposes on port 8080)
npm run docker:run
```

The Dockerfile uses `node:18-alpine`, installs NGINX, copies the app, and starts both Node.js (on internal port 3000) and NGINX (on `$PORT`) in a single container. NGINX configuration is generated at runtime via `envsubst` to inject the `$PORT` environment variable.

### NGINX Rate Limiting

Requests to `/threads` and nested routes are rate-limited to **90 requests per minute** per IP (`limit_req_zone`). Requests exceeding this threshold receive a `429 Too Many Requests` response with the `X-Rate-Limit: ACTIVE` header. All other routes are proxied without rate limiting.

```nginx
limit_req_zone $binary_remote_addr zone=threads_limit:10m rate=90r/m;

location ~ ^/threads(/.*)?$ {
  add_header X-Rate-Limit "ACTIVE" always;
  limit_req zone=threads_limit;
  limit_req_status 429;
  proxy_pass http://127.0.0.1:3000;
}
```

---

## Performance Highlights

- ✅ **Perfect score 5/5** — all automated Dicoding review criteria met
- 🧪 **100% test coverage** — 53 test files (unit, integration, functional)
- ⚡ **N+1 eliminated** — like counts fetched in a single batched `IN (...) GROUP BY` query for the entire thread
- 🔒 **DB-level duplicate prevention** — `UNIQUE(comment_id, user_id)` constraint on `comment_likes` enforces one-like-per-user at the database layer, independent of application logic
- 🔄 **CI/CD pipeline** — zero manual deployment steps; every push to master triggers the full build-migrate-test-deploy sequence
- 🐳 **Single-container deployment** — Node.js + NGINX shipped as one Docker image; deployable to Railway, Render, or any container host with a `$PORT` environment variable

---

## What I Learned

### 1. Clean Architecture Makes Testing Trivial

Every use case in this project can be fully tested with plain JavaScript objects — no database connection, no HTTP server, no file system. This is because use cases depend only on repository *interfaces*, not implementations. The test substitutes mock repositories that return predetermined values. If the test passes, the use case logic is correct regardless of which database backs it in production.

### 2. The Dependency Inversion Principle in Practice

`ThreadRepository` in `Domains/` is an abstract class with methods that throw `NotImplementedError`. `ThreadRepositoryPostgres` in `Infrastructures/` is the concrete implementation. The use case imports `ThreadRepository` — the abstraction — never `ThreadRepositoryPostgres`. This means the use case is isolated from PostgreSQL by design, not by convention.

### 3. Soft Delete Is a User Experience Decision, Not a Database Decision

Deleted comments and replies remain in the database. Whether to show their content is a use case decision made at the application layer:

```javascript
content: c.isDelete ? '**komentar telah dihapus**' : c.content
```

This preserves thread coherence — replies to deleted comments still make sense in context — and allows data recovery if needed. The alternative (hard delete) would leave orphaned replies pointing at nothing.

### 4. N+1 Is a Design Problem, Not a Performance Problem

The naive implementation of `GetThreadDetailUseCase` would fetch like counts per comment: one query per comment = N+1 round trips. The batched approach — fetch all like counts for all comments in a single query keyed by an `IN` clause — reduces this to exactly one round trip regardless of how many comments exist. The decision was made at the use case design stage, not after profiling.

### 5. The DI Container Is the Architecture's Composition Root

`container.js` is the one place where abstractions are bound to implementations. Every other file in the codebase imports interfaces. This means the entire infrastructure can theoretically be swapped — PostgreSQL to MongoDB, bcrypt to argon2, Hapi to Express — by changing `container.js` and the `Infrastructures/` implementations, with zero changes to `Domains/` or `Applications/`.

---

## License

This project was built as a Dicoding course submission and is open for educational use.

---

*Submission: Menjadi Back-End Developer Expert dengan JavaScript — Stage 7 (Capstone) of Backend JavaScript Learning Path, Dicoding 2026*
*Reviewed Score: ⭐ 5/5 — Perfect*