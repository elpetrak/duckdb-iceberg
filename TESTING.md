# Running duckdb-iceberg Tests

How to run the extension's tests, what happens when a test runs, and where the pieces live. All commands run from the root of a `duckdb-iceberg` checkout after a build (`make release`). Rebuild after changing code; the test binary doesn't rebuild itself.

## Running a Test

Tests that need no services, in `test/sql/local/` (including `iceberg_scans/` and `partitioning/`) but outside the two folders below:

```
./build/release/test/unittest test/sql/local/iceberg_fixed_type.test
```

Tests that need a catalog, in `test/sql/local/catalog_test_config_setup/catalog_agnostic/` and `test/sql/local/native_catalog/`. The in-process mock catalog provides one, so they still need no services:

```
./build/release/test/unittest --test-config test/configs/mock_native.json \
  test/sql/local/catalog_test_config_setup/catalog_agnostic/create/test_create_table_as.test
```

Useful variations:

- **A whole folder:** quote the path and add `*`, for example `"test/sql/local/catalog_test_config_setup/catalog_agnostic/create/*"`.
- **Following along:** open the `.test` file next to the run. Each `query` and `statement` block is SQL you can paste into the DuckDB shell (`./build/release/duckdb`).

The last line of the output is the result:

- `All tests passed (N assertions in M test cases)`
- a failure, showing the SQL, the expected result and the actual result
- `Skipped tests for the following reasons:` followed by the reason. A skip such as `require-env DUCKDB_ICEBERG_HAVE_GENERATED_DATA` means the test needs data or setup that isn't available, so the feature is **not checked**.

## Trying Things Interactively

`mock_native.json` only works inside `unittest`. For an interactive session in the DuckDB shell, start the Python mock catalog. It needs no Docker and listens on `127.0.0.1` only:

```
make mock
./build/release/duckdb
```

```sql
ATTACH '' AS lake (TYPE ICEBERG, URI 'http://127.0.0.1:19134', AUTHORIZATION_TYPE 'none', ACCESS_DELEGATION_MODE 'none');
CREATE SCHEMA IF NOT EXISTS lake.default;
```

Stop it with `make mock-stop`.

Local tables in `data/persistent/` need no catalog at all:

```sql
SELECT * FROM iceberg_scan('data/persistent/<table>/.../metadata/<version>.metadata.json');
```

## What Happens When a Test Runs

Running `./build/release/test/unittest [--test-config X.json] <file.test>` goes through these steps. The source is DuckDB's test runner, `duckdb/test/sqlite/sqllogic_test_runner.cpp` and `duckdb/test/helpers/test_config.cpp`.

### 1. The runner starts

`unittest` is DuckDB's own test program, built with the iceberg extension inside it. It reads the arguments and finds the matching `.test` files. If `--test-config` is given, it loads that JSON once. For example, `test/configs/mock_native.json` contributes:

- **`extends: mock.json`**: inherits the skip list and settings from `test/configs/mock.json`, the Python mock's config. Paths in `extends` are relative to the config that names them.
- **`statically_loaded_extensions`**: `iceberg`, `avro`, `parquet`, `httpfs`, `icu` and `iceberg_mock` (the in-process fake catalog).
- **`on_init`**: SQL to run before every test (step 3).
- **`skip_tests`**: test files to skip for this catalog, each with a reason.
- **`test_env`**: variables the tests can check, such as `CATALOG_TEST_CONFIG_SETUP=mock`.

### 2. Each test file gets a fresh database

Each file starts with a new, empty in-memory DuckDB, so tests can't see each other's tables. If the file is on the config's skip list, it stops here and is reported as skipped.

### 3. `on_init` runs on every new connection

With `mock_native.json` this is:

```sql
CALL iceberg_test_catalog_init('{TEMP_DIR}/iceberg-native');   -- start the in-process catalog in a temp dir
ATTACH '' AS my_datalake (TYPE ICEBERG, URI 'http://iceberg-mock.invalid', ...);
CREATE SCHEMA IF NOT EXISTS my_datalake.default;
```

The `.invalid` URI never reaches the network: `iceberg_mock` answers the REST calls inside the process. That's why the `catalog_agnostic/` and `native_catalog/` tests need no services. With `mock.json`, the URI is the real local server instead, `http://127.0.0.1:19134`. Without `--test-config`, nothing runs here and there's no `my_datalake`, which is why the `catalog_agnostic/` tests need a config.

### 4. The file runs top to bottom

`.test` files are sqllogictest scripts. Each block is one of these:

| Block | What it does |
|---|---|
| `require iceberg` / `require-env X` | Skips the whole file if an extension or an environment variable is missing |
| `include test/sql/init_requires` | Pastes in another file, here the four `require` lines shared by all iceberg tests |
| `statement ok` + SQL | The SQL must succeed |
| `statement error` + SQL + `----` + message | The SQL must fail, and the error must match the message (or the `<REGEX>:`) |
| `query IIII` + SQL + `----` + rows | Runs the query and compares every row with the expected values. One `I` per column; `rowsort` before the letters ignores row order |

Placeholders such as `{WORKING_DIRECTORY}` or `{TEMP_DIR}` are replaced before each statement runs.

### 5. The result

The first failing block stops that file, and the runner prints the SQL, what was expected and what came back. If nothing fails, it prints `All tests passed (N assertions in M test cases)`, where each `statement` and `query` counts as one assertion. Skipped files are listed with their reason. Temp directories are cleaned up afterwards.

For example, in `test/sql/local/iceberg_scans/iceberg_schemas.test`, `include` loads the extensions, each `query IIIIIII` runs `iceberg_schemas(...)` and compares the seven columns, and the final `statement error` checks that a non-Iceberg table fails with "not an Iceberg table".

## Writing a Test

Sources: `duckdb/AGENTS.md`, `duckdb/CONTRIBUTING.md`, the repository's `AGENTS.md` and `test/sql/local/catalog_test_config_setup/README.md`.

### DuckDB's generic structure (core and extensions)

**1. Prefer sqllogictest.** DuckDB strongly prefers `.test` files over C++ tests. C++ is for things SQL can't express, such as concurrent connections. Framework docs: https://duckdb.org/dev/testing.

**2. A three-line header**, followed by a blank line:

```
# name: test/sql/local/iceberg_scans/iceberg_schemas.test
# description: iceberg_schemas lists the top-level fields of every schema of a table
# group: [iceberg_scans]
```

- `name` is the file's path from the repository root.
- `group` is the name of the parent folder, in brackets.
- `description` is one line saying what the test checks.

**3. Requirements next:** `require <extension>` and `require-env VAR`, so the file is skipped, not failed, when something is missing.

**4. The body:** `statement ok`, `statement error` and `query …` blocks, separated by blank lines (see the table in step 4 above). Errors are matched with `----` plus the message or a `<REGEX>:`.

**5. Naming and speed:**

- Name files `test_<what>.test`, or after the feature (`iceberg_schemas.test`).
- Expensive tests use the `.test_slow` extension, which only runs in the full suite.
- Don't add `PRAGMA enable_verification`; DuckDB no longer uses it in new tests.

**6. What to cover:** many tests, different types (numerics, strings, nested types), and incorrect usage and errors, not only the happy path.

### duckdb-iceberg additions

- **Shared requirements:** start the body with `include test/sql/init_requires`. It loads `avro`, `parquet`, `iceberg` and `httpfs`.
- **Where the test goes:**
    - `test/sql/local/` for tests without a catalog (local files, `iceberg_scan`).
    - `test/sql/local/catalog_test_config_setup/catalog_agnostic/` for behavior all REST catalogs share. These tests start with `require-env CATALOG_TEST_CONFIG_SETUP` and use the `my_datalake` catalog that `on_init` attaches.
    - `test/sql/local/catalog_test_config_setup/catalog_specifc/<catalog>/` (the misspelling is the real folder name) for one catalog's behavior.
    - `test/sql/local/catalog_test_config_setup/catalog_interop/` when Spark or PyIceberg is involved.
- **Catalog test hygiene:**
    - Each test drops and recreates its own tables (`DROP TABLE IF EXISTS …` first).
    - Shared setup goes in `catalog_agnostic/setup/*.test_setup` and is included after the `require` lines.
    - Write timestamps with an explicit UTC offset.
    - Run catalog tests serially, because they share tables and storage.
- **Catalog limitations:** if one catalog can't do something, add the test to that catalog's `skip_tests` in `test/configs/<catalog>.json` with a reason. Don't weaken the shared test.
- **Reading metadata files directly through a REST catalog:** wrap the lookup and all the file reads in one `BEGIN` … `COMMIT`, because the catalog's storage credentials only last for that transaction.
- **Location:** `unittest` only runs test files inside the repository. A `.test` file elsewhere (for example in `/tmp`) gives `No tests ran`.

## `test/` vs. `duckdb/test/`

They belong to two different projects. `duckdb/` is a git submodule: a copy of upstream DuckDB pinned at one commit.

**`test/`: the iceberg extension's tests**

- `test/sql/`: the iceberg sqllogictests
- `test/configs/`: the iceberg configs (`mock.json`, `mock_native.json`, `polaris.json`, …)
- `test/python/`: PySpark and PyIceberg integration tests
- `test/native_catalog/`: the C++ in-process mock catalog (`iceberg_mock`)

**`duckdb/test/`: DuckDB core's own tests, plus the test runner**

- `duckdb/test/sql/`: thousands of DuckDB tests (joins, types, storage, …) that don't involve Iceberg
- `duckdb/test/sqlite/` and `duckdb/test/helpers/`: the code of the `unittest` program itself, which parses `.test` files and handles `--test-config`
- `duckdb/test/configs/`: DuckDB's own configs (block sizes, verification modes, …)

`unittest` is built from `duckdb/test/` code but runs the files in `test/`. The two also meet in `test/configs/fixture_duckdb_tests.json`, which runs parts of DuckDB's own `duckdb/test/sql/` suite against an Iceberg catalog.

Nothing under `duckdb/` should be edited from this repository. Changes there belong upstream in DuckDB.
