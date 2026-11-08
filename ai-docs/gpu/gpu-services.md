# GPU-backed services called from databases

[Research index](README.md) · [SQL offload](gpu-sql-offload.md) · [Vector search](gpu-vector-search.md)

These cases use GPU-backed models, containers, or services for a specific computation. Their results must be attributed to that computation rather than assumed to describe ordinary SQL or ANN execution.

Speedup figures come from authors' and vendors' publications; no local measurements were performed for this survey.

## ClickHouse

A verified example generates embeddings through an external program with a persistently loaded model and batched row transfer. The presentation compares 265 rows/s on CPU with up to 3500 rows/s on one GPU configuration. This is model acceleration, not evidence of GPU nearest-neighbor search or complete SQL execution. [ClickHouse example](https://presentations.clickhouse.com/2024-meetup-raleigh/ai/). General GPU offloading is discussed separately in a [project issue](https://github.com/ClickHouse/ClickHouse/issues/63392); the checked materials do not establish it as a universal engine capability.

## Snowflake

The verified path is GPU compute pools in Snowpark Container Services: training, inference, custom services, and batch functions. Documentation explicitly separates Python execution on a compute pool from SQL/Snowpark queries on a warehouse. A GPU container does not establish GPU execution of ordinary SQL. [Execution placement](https://docs.snowflake.com/en/developer-guide/snowflake-ml/notebooks-on-spcs), [Service Functions and SPCS](https://docs.snowflake.com/en/developer-guide/snowpark-container-services/overview).
