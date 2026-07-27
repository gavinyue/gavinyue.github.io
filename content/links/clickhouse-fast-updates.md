---
title: "How ClickHouse built fast UPDATEs for a column store"
date: 2026-07-25
tags: [clickhouse, analytics]
externalUrl: "https://clickhouse.com/blog/updates-in-clickhouse-1-purpose-built-engines"
---

Three-part series on making row updates fast in a column store: [purpose-built engines](https://clickhouse.com/blog/updates-in-clickhouse-1-purpose-built-engines) (ReplacingMergeTree and friends), [SQL-style UPDATEs](https://clickhouse.com/blog/updates-in-clickhouse-2-sql-style-updates), and [benchmarks](https://clickhouse.com/blog/updates-in-clickhouse-3-benchmarks) claiming 1,000× speedups over the old mutation path.
