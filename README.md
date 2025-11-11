# TPC-DS Tutorial Project

This is a Tutorial example of modeling a classic benchmark data warehouse into Strata. Many popular data warehouses like Snowflake provide a sample TPC-DS database that you can use with the model.

If you don't have a vendor provided database another quick alternative is to use DuckDB. They provide TPC DS as an extensions thats quite easy to setup.  [DuckDB TPC-DS Setup Guide](https://duckdb.org/docs/stable/core_extensions/tpcds)

## About TPC-DS

TPC-DS is a decision support benchmark that models several generally applicable aspects of a decision support system, including queries and data maintenance. The benchmark provides a representative evaluation of performance as a general purpose decision support system. A benchmark result measures query response time in single user mode, query throughput in multi user mode and data maintenance performance for a given hardware, operating system, and data processing system configuration under a controlled, complex, multi-user decision support workload. The purpose of TPC benchmarks is to provide relevant, objective performance data to industry users. TPC-DS enables emerging technologies, such as Big Data systems, to execute the benchmark. The TPC-DS Price/Performance metric is expressed as Price/QphDS@Size for Version 2 and Price/kQphDS@Size for Version 3.

[Learn More about the Benchmark.](https://www.tpc.org/tpcds/)
