# Contribution 1: Deleted namespaces may leave traces on filesystems

**Contribution Number:** 1  
**Student:** Yu-Wei Tseng  
**Issue:** [lakekeeper#1064 — Deleted namespaces may leave traces on filesystems](https://github.com/lakekeeper/lakekeeper/issues/1064)  
**Status:** Phase III In Progress

---

## Why I Chose This Issue

I chose this issue because it sits at the intersection of two areas I want to grow in: Rust and data lakehouse infrastructure. Lakekeeper is an Apache Iceberg REST catalog written in Rust, and working on it gives me hands-on experience with both the language and the data engineering ecosystem around Iceberg. I'm particularly interested in understanding how catalog services manage storage lifecycle — this issue is a concrete example of that, dealing with how namespace folders on filesystem-based storage (HDFS, ADLS) are cleaned up when a namespace is dropped.

The issue is also well-scoped and has clear maintainer guidance. The maintainer specified the exact acceptance criteria: delete the namespace folder only if (1) it is empty and (2) no other table in the same warehouse shares the same location. This bounded scope fits a 3–4 week contribution timeline, and the clear criteria make it straightforward to know when the fix is complete. The issue is labeled, unassigned, and the maintainer is active — all green flags from the issue selection checklist.

---

## Understanding the Issue

### Problem Description

When a namespace is created in Lakekeeper and a table is created within it, the system creates a corresponding folder on filesystem-based storage backends (HDFS, ADLS). However, when the namespace is dropped (after all tables within it have been dropped), the namespace folder is **not deleted** from the storage backend. On object stores like S3 this is invisible because prefixes only exist implicitly, but on true filesystem backends (HDFS, ADLS) the empty folder persists as a visible leftover.

### Expected Behavior

After dropping a namespace that contains no tables (and whose storage folder is empty), the corresponding folder on the filesystem storage backend should be cleaned up — provided no other table in the same warehouse shares the same location prefix.

### Current Behavior

The namespace folder remains on the filesystem even after the namespace is dropped. The `drop_namespace` code path in `server/namespace.rs` only handles:
- Deleting the namespace record from PostgreSQL
- Scheduling purge tasks for child tables (in recursive drops)
- Removing namespace and table authorizer entries

It never interacts with the storage backend to remove the namespace's folder.

### Affected Components

- `crates/lakekeeper/src/server/namespace.rs` — `drop_namespace()` (line 464) and `try_recursive_drop()` (line 709): the business logic entry points that must be extended to trigger namespace folder cleanup.
- `crates/lakekeeper/src/service/catalog_store/namespace.rs` — `NamespaceDropInfo` struct (line 329): needs to include namespace location(s) so the caller can schedule cleanup.
- `crates/lakekeeper-storage-postgres/src/namespace.rs` — `drop_namespace()` (line 651): the SQL query that gathers drop info needs to also retrieve the namespace's `location` property from `namespace_properties`.
- `crates/lakekeeper/src/server/io.rs` — `remove_all()` and `list_location()`: existing helpers for storage interaction that can be reused.
- `crates/lakekeeper/src/service/tasks/tabular_purge_queue.rs` — existing purge task pattern to follow as a model.

---

## Reproduction Process

### Environment Setup

Set up following the developer guide in `docs/docs/developer-guide.md`:

1. Start a PostgreSQL 17 container:
   ```bash
   docker run -d --name postgres-16 -p 5432:5432 -e POSTGRES_PASSWORD=postgres postgres:17
   ```
2. Create `.env` with the required environment variables:
   ```bash
   echo 'export DATABASE_URL=postgresql://postgres:postgres@localhost:5432/postgres' > .env
   echo 'export ICEBERG_REST__PG_ENCRYPTION_KEY="abc"' >> .env
   echo 'export ICEBERG_REST__PG_DATABASE_URL_READ="postgresql://postgres:postgres@localhost/postgres"' >> .env
   echo 'export ICEBERG_REST__PG_DATABASE_URL_WRITE="postgresql://postgres:postgres@localhost/postgres"' >> .env
   source .env
   ```
3. Install prerequisites: `cargo install sqlx-cli cargo-nextest`
4. Run migrations: `sqlx database create && sqlx migrate run --source crates/lakekeeper-storage-postgres/migrations`
5. Verify the build compiles: `cargo build --all-features`

### Steps to Reproduce

1. Set up the local dev environment with a filesystem-aware storage backend (use the minimal docker-compose example with Minio via `docker compose -f docker-compose.yaml -f docker-compose-build.yaml up -d --build` in the `examples/minimal` directory — Minio doesn't exhibit folder remnants like ADLS, but the code path can be verified through tests and code inspection).
2. Create a warehouse backed by the storage profile.
3. Create a namespace (e.g., `my_namespace`) — this sets a `location` property like `s3://bucket/warehouse-uuid/my_namespace-uuid/` in the namespace's properties stored in PostgreSQL.
4. Create a table within the namespace — this triggers the creation of the namespace folder on storage via the table creation path.
5. Drop the table (with `purge=true`) — the table's data folder is cleaned up via `TabularPurgeTask`.
6. Drop the namespace.
7. **Expected:** The now-empty namespace folder is removed from storage.
8. **Actual:** The namespace folder persists on storage. The `drop_namespace()` function in `server/namespace.rs` never schedules any storage cleanup for the namespace's own location.

**Code-level evidence:** In `server/namespace.rs` lines 533–577, the non-recursive drop path calls `C::drop_namespace()` (database only) and commits — no storage interaction. In the recursive path (lines 738–760), only child tables' locations are scheduled for purge via `TabularPurgeTask`. The namespace's own location is never referenced.

### Reproduction Evidence

- **Branch link:** https://github.com/dadavidtseng/lakekeeper/tree/fix-issue-1064 *(to be created)*
- **My findings:** Confirmed through code inspection that:
  - The `NamespaceDropInfo` struct (line 329 in `service/catalog_store/namespace.rs`) contains `child_namespaces`, `child_tables`, and `open_tasks` — but **no namespace location**.
  - The PostgreSQL `drop_namespace()` function (line 651 in `lakekeeper-storage-postgres/src/namespace.rs`) queries for child namespaces, tabulars, and tasks — but **does not query `namespace_properties`** for the `location` key.
  - The `try_recursive_drop()` function (line 738) only schedules `TabularPurgeTask` for child tables, not for namespace folders.
  - By contrast, the table drop path in `server/generic_tables/drop.rs` correctly schedules `TabularPurgeTask` with the table's location for storage cleanup.

---

## Solution Approach

### Analysis

The root cause is straightforward: the namespace drop code path was never wired up to clean the namespace's storage location. The table drop path has a well-established pattern for storage cleanup (schedule a `TabularPurgeTask` that calls `remove_all()`), but this pattern was simply never applied to namespace folders.

The maintainer specified two safety conditions:
1. The folder must be empty (no files remaining).
2. No other table in the same warehouse uses the same location prefix.

### Proposed Solution

After dropping the namespace from the database, add a step that:
1. Retrieves the namespace's `location` from its properties.
2. Checks that the location is empty on the storage backend (using `list_location()`).
3. Checks that no other active tabular in the same warehouse shares the same location prefix.
4. If both checks pass, removes the namespace folder (using `remove_all()` or a targeted directory delete).

This cleanup should happen **after** the database transaction commits (same pattern as table purge tasks) to avoid deleting storage for a namespace whose DB deletion might roll back.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** When a namespace is dropped, the corresponding folder on filesystem-based storage backends (HDFS, ADLS) is not cleaned up. The fix must delete the folder only if it is empty and not shared by any other table in the warehouse.

**Match:** The table purge pattern in `service/tasks/tabular_purge_queue.rs` is the closest existing solution. It:
- Receives a location from the drop info
- Gets the warehouse's FileIO
- Calls `remove_all()` on the location

For namespace cleanup, we can either reuse `TabularPurgeTask` or create a lightweight inline cleanup (since namespace folders should already be empty, unlike table folders which contain data files).

**Plan:**
1. Modify `NamespaceDropInfo` in `service/catalog_store/namespace.rs` to include `namespace_locations: Vec<(NamespaceId, Location)>` — the location of the dropped namespace (and child namespaces in recursive drops).
2. Update the SQL query in `lakekeeper-storage-postgres/src/namespace.rs` `drop_namespace()` to extract the `location` from `namespace_properties` JSON for the dropped namespace (and child namespaces).
3. In `server/namespace.rs`, after the DB transaction commits:
   - For each namespace location, use the warehouse's `storage_profile.file_io()` to get a storage handle.
   - Call `list_location()` to check if the folder is empty.
   - Query whether any remaining tabular in the warehouse shares the location prefix.
   - If both conditions are satisfied, call `remove_all()` to delete the folder.
4. Add unit tests in `crates/lakekeeper-integration-tests/tests/drop_recursive.rs` (or a new test file) that verify:
   - Namespace folder is deleted when empty and unshared.
   - Namespace folder is NOT deleted when non-empty.
   - Namespace folder is NOT deleted when another table shares the location.
5. Run `cargo nextest run --all-features` to confirm no regressions.

**Implement:** https://github.com/dadavidtseng/lakekeeper/tree/fix-issue-1064 *(work will begin in Phase III)*

**Review:** Self-review checklist:
- PR title follows [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) format (e.g., `fix: clean up namespace folder on filesystem after drop`)
- Run `just check-clippy` and `just fix-format` before push
- Run `just sqlx-prepare` if any SQL changes are made
- Verify against the developer guide's CI requirements

**Evaluate:**
- Manual: Reproduce the steps above and verify the namespace folder is removed after drop.
- Automated: New test cases covering the three scenarios (empty+unshared → deleted, non-empty → kept, shared location → kept).
- Regression: Full test suite passes (`cargo nextest run --all-features`).

---

## Testing Strategy

### Integration Tests (all passing)

- [x] `test_drop_empty_namespace_cleans_up_folder` — verifies that dropping a namespace with an empty storage folder triggers cleanup
- [x] `test_drop_namespace_with_nonempty_folder_keeps_folder` — verifies that dropping a namespace whose storage folder still contains files does NOT delete the folder
- [x] `test_recursive_drop_removes_namespace` — verifies that recursive drop with purge removes the namespace from the catalog; storage cleanup is best-effort and depends on async purge task completion

Tests are located in `crates/lakekeeper-integration-tests/tests/namespace_storage_cleanup.rs` and use `#[sqlx::test]` with `MemoryProfile` for in-memory storage assertions.

---

## Implementation Notes

### Week 1 Progress (June 17)

**What I built:**

1. Extended `NamespaceDropInfo` struct with a `namespace_locations: Vec<(NamespaceId, Location)>` field to carry each dropped namespace's storage location through the drop pipeline.

2. Updated the PostgreSQL `drop_namespace()` SQL query to fetch `namespace_properties->>'location'` for the target namespace and all child namespaces. Added two parallel arrays (`dropped_ns_ids`, `dropped_ns_locations`) to the CTE result, with corresponding type annotations and `.sqlx` cache update.

3. Added `try_cleanup_namespace_locations()` helper function in `server/namespace.rs` that performs best-effort storage cleanup after a namespace drop:
   - Gets storage IO via the warehouse's storage profile and secret
   - For each namespace location, checks if the folder is empty using `is_empty()`
   - If empty, removes the folder using `remove_all()`
   - Errors are logged and swallowed — cleanup must not fail the drop operation

4. Wired the cleanup into both code paths:
   - Non-recursive drop: captured `drop_info` return value and added cleanup call after commit
   - Recursive drop: added `secret_store` parameter to `try_recursive_drop` and added cleanup call after authorizer cleanup

5. Added 3 integration tests in a new test file `namespace_storage_cleanup.rs`.

**Challenges faced:**
- Git CRLF line-ending issues on Windows caused phantom diffs across 861 files. Resolved with `git config core.autocrlf input` and `git checkout -- .`.
- The sqlx compile-time query verification uses offline `.sqlx/` cache files. Adding new columns to the SQL required computing the new query hash and manually creating the updated cache JSON file, then removing the old one.
- The sandbox environment couldn't run `git commit` due to `.git/index.lock` permission restrictions — all git operations had to be run from the local terminal.

### Code Changes

- **Files modified:**
  - `crates/lakekeeper/src/service/catalog_store/namespace.rs` — added `namespace_locations` field to `NamespaceDropInfo`
  - `crates/lakekeeper-storage-postgres/src/namespace.rs` — extended SQL CTE with namespace location queries; populated `namespace_locations` in result parsing
  - `crates/lakekeeper/src/server/namespace.rs` — added `try_cleanup_namespace_locations()` helper; wired cleanup into both drop paths; added `SecretStore` bound to `try_recursive_drop`
  - `crates/lakekeeper-integration-tests/tests/namespace_storage_cleanup.rs` — new test file with 3 integration tests
  - `.sqlx/` — replaced old query cache with updated version containing new columns

- **Key commits:**
  - [`a0d45c23`](https://github.com/dadavidtseng/lakekeeper/commit/a0d45c23) — refactor: add namespace_locations field to NamespaceDropInfo
  - [`124b38dc`](https://github.com/dadavidtseng/lakekeeper/commit/124b38dc) — feat: fetch namespace locations in drop_namespace SQL query
  - [`c40bd38c`](https://github.com/dadavidtseng/lakekeeper/commit/c40bd38c) — feat: clean up empty namespace folders on storage after drop
  - [`43ad1d1f`](https://github.com/dadavidtseng/lakekeeper/commit/43ad1d1f) — fix: add type annotations to SQL arrays and update sqlx cache
  - [`ebbd5271`](https://github.com/dadavidtseng/lakekeeper/commit/ebbd5271) — test: fix unused import and recursive drop test assertion

- **Branch:** https://github.com/dadavidtseng/lakekeeper/tree/fix-issue-1064

---

## Pull Request

**PR Link:** *(To be created in Phase IV)*

**Status:** Ready for PR submission

---

## Learnings & Reflections

*(To be filled in Phase IV)*

---

## Resources Used

- [Lakekeeper Issue #1064](https://github.com/lakekeeper/lakekeeper/issues/1064)
- [Lakekeeper Repository](https://github.com/lakekeeper/lakekeeper)
- [Lakekeeper Developer Guide](https://github.com/lakekeeper/lakekeeper/blob/main/docs/docs/developer-guide.md)
- [Conventional Commits Specification](https://www.conventionalcommits.org/en/v1.0.0/)
