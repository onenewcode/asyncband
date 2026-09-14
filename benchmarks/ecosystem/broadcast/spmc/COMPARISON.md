<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# `broadcast::spmc` vs other broadcast channels

One producer. `broadcast::spmc` senders are not cloned. `broadcast::mpmc` is the same 1-producer workload. `async-broadcast` is lossless bounded (overflow off). `tokio::sync::broadcast` is only in the unbounded set, with capacity equal to the batch so the run stays on the non-blocking path rather than `Lagged`.

## Environment

| Item            | Value                                                        |
| --------------- | ------------------------------------------------------------ |
| Machine         | Apple M4, 10 cores, arm64                                    |
| OS              | macOS 26.5.2                                                 |
| rustc           | 1.97.0-nightly (f964de49b 2026-05-07)                        |
| Command         | `cargo bench -p benchmarks --bench ecosystem -- broadcast::spmc` |
| Batch           | 4096 messages                                                |
| Tokio scheduled | 4 worker threads                                             |
| Cell            | `divan` median of one run                                    |

`Asyncband` in the tables is `broadcast::spmc`. `AsyncbandMpmc` is `broadcast::mpmc` with one sender.

## Bounded (lossless wait-at-capacity)

Tokio is not in this table: it overwrites at capacity.

### Round trip (1 producer, 1 receiver)

| Bench              | async-broadcast | spmc    | mpmc 1P |
| ------------------ | --------------- | ------- | ------- |
| `try_round_trip`   | 25.2 ns         | 21.5 ns | 33.5 ns |
| `ready_round_trip` | 40.1 ns         | 23.5 ns | 42.7 ns |

### Native threads (`concurrent`)

Lower time is better. 4096 messages.

| Shape            | async-broadcast | spmc     | mpmc 1P  |
| ---------------- | --------------- | -------- | -------- |
| cap 1 / 1 recv   | 15.0 ms         | 8.71 ms  | 15.5 ms  |
| cap 1 / 8 recv   | 119 ms          | 87.8 ms  | 85.1 ms  |
| cap 1 / 32 recv  | 528.2 ms        | 352.9 ms | 237.3 ms |
| cap 64 / 1 recv  | 440 µs          | 364 µs   | 534 µs   |
| cap 64 / 8 recv  | 10.2 ms         | 7.76 ms  | 14.2 ms  |
| cap 64 / 32 recv | 38.0 ms         | 44.8 ms  | 177.8 ms |

### Tokio 4-worker (`scheduled`)

| Shape            | async-broadcast | spmc    | mpmc 1P |
| ---------------- | --------------- | ------- | ------- |
| cap 1 / 1 recv   | 1.21 ms         | 653 µs  | 866 µs  |
| cap 1 / 8 recv   | 13.9 ms         | 3.81 ms | 5.20 ms |
| cap 1 / 32 recv  | 66.6 ms         | 24.1 ms | 29.1 ms |
| cap 64 / 1 recv  | 233 µs          | 175 µs  | 246 µs  |
| cap 64 / 8 recv  | 2.17 ms         | 558 µs  | 2.28 ms |
| cap 64 / 32 recv | 6.53 ms         | 1.36 ms | 8.07 ms |

## Unbounded (non-blocking batch)

Tokio and async-broadcast are given capacity 4096 so the batch fits without lag or wait.

### Round trip

| Bench              | async-broadcast | tokio   | spmc    | mpmc 1P |
| ------------------ | --------------- | ------- | ------- | ------- |
| `try_round_trip`   | 25.2 ns         | 17.0 ns | 7.17 ns | 34.8 ns |
| `ready_round_trip` | 41.6 ns         | 21.7 ns | 11.8 ns | 37.0 ns |

### Fan-out (publish 4096, then every subscription drains)

| Receivers | async-broadcast | tokio   | spmc    | mpmc 1P |
| --------- | --------------- | ------- | ------- | ------- |
| 1         | 123 µs          | 128 µs  | 45.2 µs | 159 µs  |
| 2         | 221 µs          | 151 µs  | 89.4 µs | 258 µs  |
| 4         | 398 µs          | 163 µs  | 110 µs  | 667 µs  |
| 8         | 891 µs          | 220 µs  | 157 µs  | 1.56 ms |
| 32        | 3.21 ms         | 559 µs  | 396 µs  | 5.54 ms |

## Reading

Native-thread bounded waits use a shared condvar (`recv_blocking` / `send_blocking`) instead of one async waker per parked task. Cap 64 / 32 receivers is `44.8 ms` vs `async-broadcast` `38.0 ms`. Cap 1 / 32 is `352.9 ms` vs 1-producer mpmc `237.3 ms`. Scheduled and round-trip cells stay ahead of the lossless peers.

Unbounded send publishes without the waiter mutex when receivers exist. Fan-out at 32 receivers is `396 µs` vs Tokio’s lossy ring at `559 µs`. Unbounded try/ready round-trip is `7.17 ns` / `11.8 ns` vs Tokio `17.0 ns` / `21.7 ns`. The leftover bounded-vs-unbounded gap is wait-at-capacity parking, not the drain path.
