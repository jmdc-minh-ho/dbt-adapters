# Custom Build

Base: `e008d8f6f` (upstream/main HEAD = #1637 の boto3 移行マージコミットそのもの。#1637 を含む安定タグはまだ無いため commit を base にする)
Date: 2026-07-02 (rebuild 18)

## Changes since rebuild 17

- **戦略転換**: base を `f820566dd` (#1984 直前に固定) から `e008d8f6f` (upstream/main HEAD) に前進。
  dbt-labs が #1984 の上に #1637 (iconara PyAthena→boto3) をマージしたことで、rebuild 17 まで前提だった
  「#1637 と #1984 が両立しない」問題が upstream 側で解決済みになった。これにより:
  - #1637 の手動マージ (旧 `iconara/direct-calls-to-athena`) が不要 → base 内包
  - #1984 (cancel in-flight queries) の明示除外が不要 → base 内包 (`Fixes-20260528-162219.yaml` も base の 1.11.0 changelog に含まれる)
  - #2000 (pin core <2.0) も base 内包
- PR #1740 / #1814 を **close** (本 rebuild 作業中に実施)。#1637 の boto3 cursor は retry 対象を
  throttling API error と `ICEBERG_COMMIT_ERROR` のみに限定しており、`ICEBERG_FILESYSTEM_ERROR` や
  query timeout 等の deterministic error は元々 retry されない = 両 PR は #1637 に機能包含。
- PR #1874 (`feat/spark-3.5-support`) を `32e2e00fd` から `df926f073` に更新:
  - `b1f17fc11` で upstream/main (#1637 含む) へ rebase。
  - `df926f073` で test stub `_StubReattachIterator` に `_release_until` / `_release_all` を追加。
    `pyspark_patches.py` が両メソッドを patch するのに stub が欠いていてブランチ単体で 8 tests が
    `AttributeError` で落ちていた (ブランチ固有バグ、組み合わせ起因ではない)。

## Included PRs

- https://github.com/dbt-labs/dbt-adapters/pull/1211 — Fix a debug log about the Athena workgroup (branch: `origin/patch-1`)
- https://github.com/dbt-labs/dbt-adapters/pull/1704 — perf(athena): cache `_get_data_catalog()` result to avoid repeated STS calls (branch: `origin/fix/lru-cache-data-catalog`)
- https://github.com/dbt-labs/dbt-adapters/pull/1705 — fix(athena): fix "connection never acquired" with `--no-populate-cache` and `threads > 1` (branch: `origin/fix/no-populate-cache-thread-connection`)
- https://github.com/dbt-labs/dbt-adapters/pull/1743 — fix(athena): handle unpartitioned models in create_table_as_with_partitions (branch: `origin/fix/athena-unpartitioned-too-many-open-partitions`)
- https://github.com/dbt-labs/dbt-adapters/pull/1749 — fix(athena): create empty target table when no partition batches found (branch: `origin/fix/athena-empty-batch-target-table`)
- https://github.com/dtaniwaki/dbt-adapters/pull/3 — feat(athena): add disable_batch_fallback config option (branch: `origin/worktree-soft-chasing-ullman`)
- https://github.com/dtaniwaki/dbt-adapters/pull/4 — feat(athena): add model-level timeout support (branch: `origin/feat/athena-model-timeout`)
- https://github.com/dbt-labs/dbt-adapters/pull/1830 — feat(athena): add build_strategy config for incremental and table materializations (branch: `origin/feat/athena-build-with-subquery`)
- https://github.com/dbt-labs/dbt-adapters/pull/1832 — feat(athena): add merge_exclude_source_columns config (branch: `origin/feat/athena-merge-select-exclude-columns`)
- https://github.com/dtaniwaki/dbt-adapters/pull/7 — feat(athena): resolve cross-account Glue catalogs in Spark Python models (branch: `origin/feat/athena-spark-cross-account-catalog`)
- https://github.com/dbt-labs/dbt-adapters/pull/1874 — feat(athena): add Apache Spark 3.5 support via Spark Connect (branch: `origin/feat/spark-3.5-support` `df926f073`)
- https://github.com/dbt-labs/dbt-adapters/pull/1881 — feat(athena): add use_iceberg_write_to config for Iceberg Python models (branch: `origin/fix/iceberg-python-writeto`)
- https://github.com/dbt-labs/dbt-adapters/pull/1990 — feat(athena): let Python models self-materialize by returning None (branch: `origin/feat/athena-python-model-skip-materialize`)
- https://github.com/dbt-labs/dbt-adapters/pull/1998 — fix(athena): use RefreshableCredentials for AssumeRole sessions (branch: `origin/fix/athena-refreshable-assume-role-credentials`)

## Already in base (no explicit merge needed)

- https://github.com/dbt-labs/dbt-adapters/pull/1221 — Add a debug log about an Athena execution error
- https://github.com/dbt-labs/dbt-adapters/pull/1636 — Fix a bucket partitioning error against many partitions
- https://github.com/dbt-labs/dbt-adapters/pull/1637 — feat(athena): Migrate dbt-athena from PyAthena to direct boto3 calls (base HEAD)
- https://github.com/dbt-labs/dbt-adapters/pull/1650 — Override check_schema_exists in Athena adapter to use Glue API
- https://github.com/dbt-labs/dbt-adapters/pull/1657 — feat(athena): Add STS AssumeRole support for cross-account access
- https://github.com/dbt-labs/dbt-adapters/pull/1784 — fix(athena): Coordinate chunk sizes in get_partition_batches
- https://github.com/dbt-labs/dbt-adapters/pull/1984 — fix(athena): cancel in-flight Athena queries when a dbt invocation is cancelled
- https://github.com/dbt-labs/dbt-adapters/pull/2000 — pin core to <2.0

## Closed / Excluded PRs

- https://github.com/dbt-labs/dbt-adapters/pull/1740 — fix(athena): exclude ICEBERG_FILESYSTEM_ERROR from outer retry (**closed**: #1637 に機能包含)
- https://github.com/dbt-labs/dbt-adapters/pull/1814 — fix(athena): skip retry of deterministic errors with configurable timeout handling (**closed**: #1637 に機能包含)

## Conflict Resolutions

すべて rerere 自動解決 + 一部手動解決。主なパターン:

- `dbt-athena/tests/unit/test_connection_manager.py` (#1874 マージ時): base の `AthenaCursor` import と
  `test_cancel_tells_the_connection_to_cancel`、#1874 の `SparkConnectSessionPool` import と
  `test_cleanup_all_terminates_only_current_invocation_sessions` を両方併存 (rerere)
- `dbt-athena/src/dbt/adapters/athena/connections.py` (#1874 マージ時): `_connection_keys` に
  `spark_connect_*` キー群と base の `work_group` / `skip_workgroup_check` を両方併存 (rerere)
- `dbt-athena/src/dbt/adapters/athena/exceptions.py` (#4 マージ時): base の `AthenaError` 階層と
  #4 の `AthenaModelTimeoutError` を両方残す (手動解決 → rerere 記録)
- `dbt-athena/src/dbt/adapters/athena/exceptions.py` (#1874 マージ時): `AthenaModelTimeoutError` と
  #1874 の `SparkSessionTerminatedError` を両方残す (手動解決 → rerere 記録)
- `.changes/1.11.0/Fixes-20260528-162219.yaml` (post-merge fixes cherry-pick 時): #1984 が base 内包に
  なったため、rebuild 17 の「削除」ではなく保持で解決

## Post-merge fixes

複数 PR の組み合わせで生じるテスト破綻を単一コミット `fix(athena): post-merge fixes for custom-build rebuild`
で補正 (前回コミット `022e22f14` を cherry-pick)。全 unit tests: 555 passed。

- `create_table_as_with_partitions`: 空バッチ check を `partitioned_by` guard の内側へ移動
- `test_adapter`: `OperationalError` → `AthenaError` (#1637 が error type を変更)
- `test_build_with_subquery` / `test_merge_exclude_source_columns`: MockAdapter に `disable_batch_fallback` kwarg 追加
- `test_python_submissions`: `SparkSessionId` / `SparkCalculationExecutionId` フィールドを expected に追加
- `test_create_table_as_with_partitions`: `config.get('partitioned_by')` を mock して partitioned 経路へ
- `test_py_save_table_as` / `test_spark_dbt_obj`: cross-account-catalog テンプレート変数用の config stub 注入

注: `test_channel.py` の stub 修正 (`_release_until` / `_release_all`) はブランチ固有バグのため
post-merge fixes には含めず、`feat/spark-3.5-support` ブランチ側 (`df926f073`) で修正済み。
