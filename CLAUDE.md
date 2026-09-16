# CLAUDE.md

Guidance for Claude Code working in this repository.

## What this is

Utopia — a bitemporal knowledge graph / enterprise world model. **One Rust binary plus one Postgres**: full-text search is embedded (Tantivy), vectors live in pgvector, and the job queue is a table. There is nothing else to run.

Read `docs/pipeline.md` first (how a document becomes a graph, and where each stage drops things), then `docs/decisions/README.md` for the reasoning behind the design.

## Layout

```
crates/utopia-core     domain models, AppError, config (all env vars are UTOPIA_*)
crates/utopia-store    sqlx repositories, migrations runner, job queue — runtime queries, not macros, so builds need no database
crates/utopia-ingest   parsers (PDF/DOCX/XLSX/CSV/HTML/…), chunker, RDF ontology import
crates/utopia-extract  LLM extraction of entities and facts
crates/utopia-reason   ontology axioms, forward-chaining derivation
crates/utopia-search   Tantivy index
crates/utopia-llm      OpenAI-compatible client
crates/utopia-server   axum HTTP API, background worker, connectors, query engines — the only binary
web/                   React 18 + TanStack Router/Query + Tailwind 4 + Vite, sigma.js for the graph
migrations/            NNNN_name.sql, forward-only
```

`utopia-server/src/api/` holds the routes; `api/mod.rs` is the router. `utopia-server/src/query_engine/` is the mounted-database layer (Postgres, Trino, Databricks, Snowflake) behind one trait.

## Commands

```bash
docker compose up -d db                              # Postgres+pgvector on 127.0.0.1:1517
cargo run -p utopia-server                           # :1516, runs migrations on startup
cd web && pnpm install && pnpm dev                   # :5173, proxies /api

# CI runs exactly this — green here is green there
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
cd web && pnpm install --frozen-lockfile && pnpm build   # build type-checks

./scripts/smoke.sh          # register → workspace → KB → permissions → job queue
./scripts/e2e_type_drift.sh # isolated DB + port, drives the type-resolution path
```

## Traps that have actually bitten this repo

**A green `cargo test --workspace` does not mean the SQL ran.** Database-backed tests begin `let Some(url) = utopia_store::test_db::url() else { return Ok(()) };` and skip silently without `UTOPIA_DATABASE_URL`. If you touched SQL under `crates/utopia-store/`, run:

```bash
UTOPIA_DATABASE_URL=postgres://utopia:utopia@localhost:1517/utopia cargo test --workspace
```

Use `test_db::url()` in new tests; never read the env var directly. CI's `migrations` job sets `UTOPIA_TEST_REQUIRE_DB=1`, which turns a missing database into a failure.

**Don't collide migration numbers.** Check the highest number on `main` before adding one. Two branches each writing `0025_` merges cleanly in git and then neither runs; CI checks for duplicates explicitly. Migrations roll forward only — no rollback, and they must be safe to replay.

**Lists that exist on both sides must have one source.** `SourceKind` in `utopia-core/src/models.rs` (strum-derived) and `web/src/sourceKinds.ts` are cross-checked by a `utopia-store` test. New job types are registered in the `dispatch` match in `utopia-server/src/main.rs`.

**Every workflow declares its own `permissions:`.** The repository default is read-and-write; omitting the block silently grants write. `contents: read` for anything that only builds or tests.

## Conventions

- **Language split**: code comments are Chinese; UI strings, README, decision records and server error text are English. Follow whatever the file around you does.
- **Comments explain why, and record the trap that was hit** ("the first version used OR, and the Elon Musk article then produced a snapshot every 6KB"). A comment restating the code will be asked to go.
- **UI strings go in i18n**, both `web/src/i18n/en.ts` and `zh.ts`. No hard-coded strings in components. `S` is resolved once at module load; changing language reloads the page.
- **Server errors carry a stable `code`**; `web/src/api.ts` words it from `S.err` at one choke point, falling back to the server's English sentence.
- **Data model, ontology contract or public API changes need an ADR** in `docs/decisions/` before the code. The PR that implements a record updates its status line.
- **Commit messages: one English sentence stating the motivation.** No body. Skim `git log` for the register.
- **Every commit needs DCO sign-off** — `git commit -s`.
- **Branch off `dev` and open PRs against `dev`, never `main`.** `main` is maintainers-only, merged from `dev`.
