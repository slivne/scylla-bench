# scylla-bench

scylla-bench is a benchmarking tool for [Scylla](https://github.com/scylladb/scylla) written in Go. It aims at minimising the client overhead and provide a wide range of test scenarios.

## Install

```
go get github.com/pdziepak/scylla-bench
```

## Usage

### Schema adjustments

The default scylla-bench schema for regular columns looks like this:

```
CREATE TABLE IF NOT EXISTS scylla_bench.test (
    pk bigint,
    ck bigint,
    v blob,
    PRIMARY KEY(pk, ck)
) WITH compression = { }
```

scylla-bench allows configuring the number of partitions, number of rows in a partition and the size of a single row. This is done using flags `-partition-count`, `-clustering-row-count` and `-clustering-row-size` respectively.

### Modes

scylla-bench can operate in one of three modes (flag `-mode`) which basically determine what kind of requests are sent to the server. Some of the modes allow additional, further configuration.

#### Write mode (`-mode write`)

The behaviour in this mode differs depending on the configured number of rows per requests. If `-rows-per-request` is set to 1 (default) scylla-bench sends simple INSERT requests like this:

```
INSERT INTO scylla_bench.test (pk, ck, v) VALUES (?, ?, ?)
```

Otherwise, writes are sent in unlogged batches each containing at most `rows-per-request` insertions. All writes in a single batch refer to the same partition. The consequence of this is that in some configuration the number of rows written in a single requests can be actually smaller than the set value (e.g. `-clustering-row-count 2 -rows-per-request 4`).

#### Counter update mode (`-mode counter_update`)

Counter updates are written to a separate column family:
```
CREATE TABLE IF NOT EXISTS scylla_bench.test_counters (
    pk bigint,
    ck bigint,
    c1 counter,
    c2 counter,
    c3 counter,
    c4 counter,
    c5 counter,
    PRIMARY KEY(pk, ck)
) WITH compression = { }
```

Each requests updates all five counters in a row and only one row per request is supported:

```
UPDATE scylla_bench.test_counters SET c1 = c1 + 1, c2 = c2 + 1, c3 = c3 + 1, c4 = c4 + 1, c5 = c5 + 1 WHERE pk = ? AND ck = ?
```

#### Read mode (`-mode read`)

Read mode is essentially split into four sub-modes and offers most configurability. The default requests resemble single partition paging queries, there is a lower bound of clustering keys and a limit which is can be adjusted using flag `rows-per-request`:

```
SELECT * FROM scylla_bench.test WHERE pk = ? AND ck >= ? LIMIT ?
```

It is possible to send request without a lower bound if flag `-no-lower-bound` is set:

```
SELECT * FROM %s.%s WHERE pk = ? LIMIT ?
```

Limit can be replaced by upper bound (flag `-provide-upper-bound`). In this case scylla-bench will choose the upper bound so that the expected number of rows equals the one specified by `rows-per-request`.

```
SELECT * FROM %s.%s WHERE pk = ? AND ck >= ? AND ck < ?
```

Finally, scylla-bench can send a request with an IN restriction (flag `-in-restriction`). Again, the number of requested clustering keys will equal `rows-per-request`.

```
SELECT * from %s.%s WHERE pk = ? AND ck IN (?, ...)
```

#### Counter read mode (`-mode counter_read`)

Counter read mode works in exactly the same as regular read mode (with the same configuration flags available) except that it reads data from the counter table `scylla_bench.test_counters`.

### Workloads

The second very important part of scylla-bench configuration is the workload. While mode chooses what kind of requests are to be sent to the cluster the workload decides which partitions and rows should be the target of these requests.

#### Sequential workload (`-workload sequential`)

This workload sequentially visits all partitions and rows in them. If the concurrency is larger than one then the hole population is split evenly between goroutines. Sequential workload allows specifying the offset of the first partition in the population (flag `-partition-offset`) to enable sequential population of the database by multiple clients. For example, if we have three simultaneously running scylla-bench processes:

1. `scylla-bench -workload sequential -mode write -partition-count 5`
1. `scylla-bench -workload sequential -mode write -partition-count 5 -partition-offset 5`
1. `scylla-bench -workload sequential -mode write -partition-count 5 -partition-offset 10`

The first loader will write partitions [0, 5), the second [5, 10) and the third [10, 15).

The sequantial workload is useful for initial population of the database (in write mode) or warming up the cache for in-memory tests (in read mode).

#### Uniform workload (`-workload unifrom`)

Uniform workload chooses the partition key and clustering key randomly with a uniform distribution.

scylla-bench requires that the maximum duration of the test is specified when running with uniform workload (e.g. `-duration 1h`).

#### Time series workload (`-workload timeseries`)

Time series workload is the most complex one and behaves differently depending whether scylla-bench is run in write or read mode.

##### Write mode

In write mode time series workload divdes the set of partitions between all goroutines (it is required that `-partition-count` >= `-concurrency`). Then each goroutine prepends to its partitions (partitions are chosen in a round-robind manner) new rows. Newer rows have smaller clustering keys than the older ones.

Once the partition reches `clustering-row-count` rows the goroutine will switch to a new partiton key. This means that the total partition count will be larger than `-partition-count`, since in time series workload that flag specifies only the number of partitions to which data is concurrently written.

The rate at which rows depends on `-max-rate` flag which must be specified in this workload. Since `-max-rate` sets the total maximum request rate of the whole client the rate at which rows will be appended to a single partition may be lower. The acutal per-partition is printed in scylla-bench configuration as `Write rate` (it is `concurrency / partition-count`). scylla-bench also prints "Start timestamp" which is necessary if there is a time series read load running.

##### Read mode

Time series workload in read mode is supposed to be run simultanously with time series writes. It requires specifying the start timestamp `-start-timestamp` and per-partition write rate `-write-rate` both of which are printed by scylla-bench running in write mode.

The time series workload in read mode chooses partition and clustering keys randomly from the range that has been written up to this point (using start timestamp and write rate). The distribution can be either uniform (flag `-distribution uniform`) or half-normal with the latest rows being most likely (`-distribution hnormal`).

Note that if the effective write rate is lower than the specified one the reader may attempt to read rows that are not yet present in the database. However, because `-max-rate` doesn't just limit the rate but tries to make the average op/s equal the specified values it will be able to recover from small periodic dips in write throughput.

### Other notable options

* `-concurrency` sets the number of goroutines used by the benchmark. The higher concurrency the higher internal client overheads, in some cases it may be better to use more than one client process instead of further increasing the concurrency.
* `-max-rate` set the expected rate of requests. The benchmark will try to reach this average which means that it may actually send more request per second if there was a period during which the throughput was lower than expected.
* `-connection-count` sets the number of connections.
* `-replication-factor` sets the replication factor of scylla-bench keyspace (default: 1).
* `-timeout` sets client timeout (default: 5s).
* `-client-compression` enables or disables client compression (default: enabled).

## Examples

1. Sequential write to populate the database: `scylla-bench -workload sequential -mode write -nodes 127.0.0.1`
2. Read test: `scylla-bench -workload uniform -mode read -concurrency 128 -duration 15m -nodes some_node`
3. Read latency test: `scylla-bench -workload uniform -mode read -duration 15m -concurrency 32 -max-rate 32000 -nodes 192.168.8.4`
4. Counter write test: `scylla-bench -workload uniform -mode counter_update -duration 30m -concurrency 128`


