# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Spellbook is Dune Analytics' open-source interpretation layer for blockchain data. It's a **dbt monorepo** that transforms raw blockchain data into clean, usable datasets ("spells") using SQL + Jinja2 templating on DuneSQL (Trino-based). Contributions are selective — open a GitHub issue with prefix `[CONTRIBUTION]` before writing code (except small bug fixes, which can go straight to a PR).

## Repository Structure

```
spellbook/
├── dbt_subprojects/        # Independent dbt sub-projects
│   ├── daily_spellbook/    # Default location for new spells (daily refresh)
│   ├── hourly_spellbook/   # Promoted spells, higher frequency (Dune team approval required)
│   ├── dex/                # DEX and DEX aggregator trading data (dex.trades)
│   ├── nft/                # NFT-related models
│   ├── solana/             # Solana-specific models
│   └── tokens/             # Token metadata, transfers, balances
├── dbt_macros/             # Shared macros (used by all sub-projects)
│   ├── dune/               # Core Dune/Trino macros
│   ├── shared/             # Cross-project shared macros (Balancer, EVM chains, etc.)
│   └── generic-tests/      # Custom dbt test macros
├── sources/                # 170+ source YAML files (raw/decoded table definitions)
│   ├── _base_sources/      # Base EVM and other source definitions
│   ├── _datasets/          # Dataset-level sources
│   ├── _sector/            # Sector-level sources
│   ├── _subprojects_outputs/ # Cross-subproject output references
│   └── <protocol>/         # Per-protocol source YAML files
├── scripts/                # Dev utilities (source generators, schema checks, etc.)
├── docs/                   # Design principles and contributor documentation
├── git_scripts/            # Git automation helpers
└── models/                 # Legacy top-level models (minimal)
```

Each sub-project is self-contained with its own `dbt_project.yml`, `profiles.yml`, `models/`, `macros/`, `seeds/`, and `tests/`. All sub-projects share `dbt_macros/` and `sources/` from the repo root.

## Commands

### Setup
```bash
pipenv install          # Create virtual environment (Python 3.9+, dbt-trino 1.9.0)
pipenv shell            # Activate environment
```

### Build & Compile (must be run from a sub-project directory)
```bash
cd dbt_subprojects/<subproject>/
dbt clean               # Clean old artifacts (target/, dbt_packages/)
dbt deps                # Pull dbt dependencies
dbt compile             # Compile Jinja/SQL → plain SQL in target/
```

`dbt compile` is the primary local validation tool. Copy compiled SQL from `target/` (mirrors directory structure) and test on dune.com. The `profiles.yml` in each sub-project must be present — always run dbt commands from the sub-project root.

### Testing Models
```bash
# Run dbt tests for a model
dbt test --select @model_name

# Run dbt tests for modified models
dbt test --select state:modified

# Compile a specific model (validates Jinja syntax)
dbt compile --select model_name
```

After `dbt compile`, test the compiled SQL directly on [dune.com](https://dune.com). The compiled output is in `target/compiled/<subproject>/models/<path>/<model_name>.sql`.

### Pre-push Hooks (optional, currently slow)
```bash
pre-commit install --hook-type pre-push
pre-commit run --hook-stage manual    # Manual run on staged files
```

Note: Pre-push hooks run `dbt compile` which is slow due to project size. The same checks run in GitHub Actions on PRs, so hooks are optional.

### CI
PRs trigger GitHub Actions (`dbt slim ci`) that test only modified models using state comparison. Test tables are named:
```
test_schema.git_dunesql_<7-char-commit-hash>_<table_name>
```
Tables are available for ~24 hours. Find exact names in the `dbt run initial model(s)` step of the CI logs.

## DuneSQL / Trino SQL Rules

### Data Types
- `block_date` → `DATE` (e.g., `DATE '2025-10-08'`)
- `block_time` → `TIMESTAMP`
- Large integers → `UINT256` or `INT256` (not `BIGINT`)
- Addresses → `VARBINARY` with `0x` prefix (never strings)
- Use `bytearray_substring(data, offset, length)` for binary slicing

### Query Rules
- **Always use explicit table aliases** — prefix every column (`t.column`, `p.column`), never bare column names
- **Use `UNION ALL`** instead of `UNION` unless deduplication is strictly needed
- **Avoid `ORDER BY`** on large result sets
- **Use `LIMIT`** during development to avoid full scans

### Partitioning and Filtering
- Filter on `block_date` (not `block_time`) for partition pruning
- Always include partition columns in WHERE and JOIN conditions
- Large tables (trades, transfers): partition on `block_month` (`date_trunc('month', block_time)`)
- Smaller tables: partition on `block_date`
- In incremental models: use `block_date` filters in non-incremental mode, `block_time` in incremental mode

### Performance
- Large table joins: use `{{ enforce_join_distribution("PARTITIONED") }}` before the query
- Larger table always goes on the **left** side of joins
- Use `dev_dates` variable to limit data in dev: `{% if var('dev_dates', false) %} AND block_date > current_date - interval '3' day {% endif %}`

## dbt Model Patterns

### Materialization
Models default to `view` (set globally in `dbt_project.yml`). Override with `config()` block when needed:
- `view` — default, fast to build
- `table` — for frequently-queried, expensive models
- `incremental` — for large, append-heavy datasets

### Incremental Model Config
```sql
{{ config(
    schema = 'dex_ethereum',
    alias = 'base_trades',
    partition_by = ['block_month'],
    materialized = 'incremental',
    file_format = 'delta',
    incremental_strategy = 'merge',
    unique_key = ['blockchain', 'project', 'version', 'tx_hash', 'evt_index'],
    incremental_predicates = [incremental_predicate('DBT_INTERNAL_DEST.block_time')]
)}}
```

Key rules:
- Always include the partition column in `unique_key`
- Use `[incremental_predicate('DBT_INTERNAL_DEST.block_time')]` for the predicate
- Use `is_incremental()` guard for incremental-only WHERE clauses:

```sql
{% if is_incremental() %}
WHERE {{ incremental_predicate('block_time') }}
{% endif %}
```

### References and Sources
```sql
-- Reference another dbt model (use filename without .sql)
{{ ref('dex_ethereum_base_trades') }}

-- Reference raw/decoded tables from sources/
{{ source('uniswap_v3_ethereum', 'UniswapV3Pool_evt_Swap') }}
```

Never hardcode table names — always use `ref()` or `source()`.

### Schema and Alias Override
```sql
{{ config(
    schema = 'dex_ethereum',   -- overrides dbt_project.yml schema
    alias = 'base_trades'      -- table name (overrides filename)
)}}
```

The resulting table in prod is `dex_ethereum.base_trades`.

### File Conventions
- One model (table/view/incremental) or one macro per file
- File name = model name (unless `alias` is set in config)
- Every model must have a corresponding entry in `_schema.yml` (or `schema.yml`) in the same directory with:
  - `description`
  - `data_tests` (at minimum `unique` + `not_null` on primary key columns)
  - `columns` with descriptions
- Directory convention: `models/<sector_or_project>/<chain>/` or `models/<sector>/<chain>/platforms/`

### Schema YAML Example
```yaml
version: 2

models:
  - name: dex_ethereum_base_trades
    meta:
      blockchain: ethereum
      sector: dex
      contributors: contributor_name
    config:
      tags: ['dex', 'ethereum', 'base_trades']
    description: "Base-level DEX trades on Ethereum"
    data_tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - tx_hash
            - evt_index
    columns:
      - name: blockchain
        description: "Blockchain name"
      - name: tx_hash
        description: "Transaction hash"
        data_tests:
          - not_null
```

## Key Macros (in `dbt_macros/`)

### `dbt_macros/dune/` — Core Dune/Trino Macros

| Macro | Signature | Purpose |
|-------|-----------|---------|
| `expose_spells_hide_trino` | `(blockchains, spell_type, spell_name, contributors)` | Makes model public on dune.com (prod only), hides from Trino datasets |
| `expose_dataset` | `(blockchains, contributors)` | Makes third-party data model public (prod only) |
| `mark_as_spell` | `(this, materialization)` | Tags model as abstraction (applied globally via post-hook) |
| `optimize_spell` | `(this, materialization)` | Runs OPTIMIZE on tables (prod only, applied globally) |
| `incremental_predicate` | `(column)` | Generates time-based incremental WHERE clause |
| `incremental_days_forward_predicate` | `(column, base_time, days_forward)` | Incremental predicate with forward-looking window |
| `enforce_join_distribution` | `(value)` | Sets `join_distribution_type` session property (use `"PARTITIONED"`) |
| `set_trino_session_property` | `(enabled, property, value)` | Sets a Trino session property |
| `is_materialized` | `(model)` | Returns true if model is table or incremental |

### `dbt_macros/shared/` — Cross-Project Macros

| Macro | Purpose |
|-------|---------|
| `all_evm_chains()` | Returns list of supported EVM chains (ethereum, optimism, arbitrum, avalanche_c, polygon, bnb, gnosis, fantom, celo, base, zksync, zora, scroll, mantle) |
| `all_op_chains()` | Returns OP Stack chain list |
| `add_tx_columns` | Adds standard transaction columns |
| `native_token_prices` | Native token price lookup |
| `balances_incremental_subset_daily` | Balances incremental pattern |

### Global Post-Hooks (Applied to All Models)
All sub-projects apply these post-hooks via `dbt_project.yml` (you don't add them manually):
1. `set_trino_session_property` — `writer_scaling_min_data_processed` (configurable via `writer_min_size`)
2. `set_trino_session_property` — `task_scale_writers_enabled = false`
3. `optimize_spell` — runs OPTIMIZE on materialized tables in prod
4. `mark_as_spell` — tags all models as abstractions in prod

### Exposing Models as Public Spells
To make a model publicly visible on dune.com, add to the model's `config()`:
```sql
{{ config(
    post_hook = '{{ expose_spells_hide_trino(
        \'["ethereum", "arbitrum"]\',
        "sector",
        "dex",
        \'["contributor_name"]\'
    ) }}'
)}}
```

Use `expose_dataset` instead for third-party/raw data models.

## DEX Sub-Project Model Hierarchy

The `dex` sub-project follows a layered model pattern for trades:

```
platforms/<protocol>_<chain>_base_trades.sql  ← raw protocol events, minimal transformation
       ↓
dex_<chain>_base_trades.sql                   ← UNION ALL of all protocols on a chain
       ↓
dex_<chain>_trades.sql                        ← enriched with token metadata, prices, tx info
       ↓
dex_trades.sql                                ← UNION ALL of all chains (top-level view)
```

All three levels use `incremental` + `merge` strategy with `block_month` partitioning.

## Supported Blockchains

**EVM chains** (in `all_evm_chains()`): ethereum, optimism, arbitrum, avalanche_c, polygon, bnb, gnosis, fantom, celo, base, zksync, zora, scroll, mantle

**Additional chains** (in dex/nft sub-projects): abstract, apechain, berachain, blast, boba, corn, flare, flow, hemi, hyperevm, ink, kaia, katana, linea, megaeth, mezo, monad, nova, opbnb, peaq, plasma, plume, ronin, sei, shape, somnia, sonic, unichain, worldchain, zkevm

**Non-EVM**: solana (separate sub-project)

## Sources Structure

Source YAML files in `sources/` define raw and decoded table schemas for all protocols:

```yaml
version: 2
sources:
  - name: uniswap_v3_ethereum          # schema name
    description: "..."
    tables:
      - name: UniswapV3Pool_evt_Swap   # table name
        columns:
          - name: contract_address
          - name: evt_block_time
          - name: evt_tx_hash
          # ...
```

Reference with `{{ source('uniswap_v3_ethereum', 'UniswapV3Pool_evt_Swap') }}`.

Sources are organized as:
- `sources/<protocol>/` — per-protocol sources (e.g., `sources/aave/`, `sources/uniswap/`)
- `sources/_base_sources/` — base EVM chain sources
- `sources/_datasets/` — Dune-managed dataset sources (prices, labels, etc.)
- `sources/_sector/` — sector-level shared sources
- `sources/_subprojects_outputs/` — references to other sub-project outputs

## Testing Workflow

1. Write/modify the SQL model
2. `dbt compile` in the relevant sub-project directory to validate Jinja/SQL syntax
3. Copy compiled SQL from `target/compiled/...` and test on dune.com
4. Add/update `_schema.yml` with description, tests, and column docs
5. Submit PR — CI creates test tables `test_schema.git_dunesql_<7-char-hash>_<table_name>`
6. Validate test vs prod by querying the test table on dune.com

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DBT_ENV_INCREMENTAL_TIME` | `"3"` | Number of time units for incremental lookback |
| `DBT_ENV_INCREMENTAL_TIME_UNIT` | `"day"` | Unit for incremental predicate (day/hour) |
| `DBT_ENV_CUSTOM_ENV_S3_BUCKET` | `"local"` | S3 bucket for CI artifacts |
| `dev_dates` | `false` | Set to `true` to limit data to last 3 days in dev |

## Common Pitfalls

- **Never use bare column names** — always alias tables and qualify columns
- **Never hardcode schema/table names** — always use `ref()` or `source()`
- **Don't filter on `block_time` for partitions** — use `block_date` for partition pruning
- **Don't add `mark_as_spell` or `optimize_spell` manually** — they're applied globally
- **`expose_spells_hide_trino` is not the same as `mark_as_spell`** — the former makes models public, the latter just tags them
- **Run dbt commands from the sub-project directory**, not the repo root
- **Each file = one model or one macro** — no multi-statement SQL files
- **Models must be SELECT statements only** — no DDL, no DML
- **`dbt_project.yml` schema assignments** are hierarchical; model-level `config(schema=...)` overrides them
