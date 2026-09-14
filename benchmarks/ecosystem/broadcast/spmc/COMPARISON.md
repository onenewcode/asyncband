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
| `try_round_trip`   | 24.3 ns         | 19.1 ns | 32.0 ns |
| `ready_round_trip` | 38.4 ns         | 22.6 ns | 43.0 ns |

### Native threads (`concurrent`)

Lower time is better. 4096 messages.

| Shape            | async-broadcast | spmc     | mpmc 1P  |
| ---------------- | --------------- | -------- | -------- |
| cap 1 / 1 recv   | 14.7 ms         | 8.41 ms  | 15.0 ms  |
| cap 1 / 8 recv   | 122 ms          | 101.5 ms | 82.5 ms  |
| cap 1 / 32 recv  | 531.4 ms        | 328.8 ms | 239.5 ms |
| cap 64 / 1 recv  | 463 µs          | 388 µs   | 525 µs   |
| cap 64 / 8 recv  | 10.0 ms         | 8.42 ms  | 14.0 ms  |
| cap 64 / 32 recv | 39.8 ms         | 46.8 ms  | 176.6 ms |

### Tokio 4-worker (`scheduled`)

| Shape            | async-broadcast | spmc    | mpmc 1P |
| ---------------- | --------------- | ------- | ------- |
| cap 1 / 1 recv   | 1.27 ms         | 666 µs  | 908 µs  |
| cap 1 / 8 recv   | 14.2 ms         | 3.72 ms | 5.05 ms |
| cap 1 / 32 recv  | 67.2 ms         | 23.6 ms | 28.9 ms |
| cap 64 / 1 recv  | 218 µs          | 175 µs  | 243 µs  |
| cap 64 / 8 recv  | 2.09 ms         | 534 µs  | 2.23 ms |
| cap 64 / 32 recv | 6.00 ms         | 1.38 ms | 7.92 ms |

## Unbounded (non-blocking batch)

Tokio and async-broadcast are given capacity 4096 so the batch fits without lag or wait.

### Round trip

| Bench              | async-broadcast | tokio   | spmc    | mpmc 1P |
| ------------------ | --------------- | ------- | ------- | ------- |
| `try_round_trip`   | 24.3 ns         | 16.8 ns | 7.11 ns | 34.0 ns |
| `ready_round_trip` | 23.8 ns         | 21.0 ns | 10.5 ns | 37.0 ns |

### Fan-out (publish 4096, then every subscription drains)

| Receivers | async-broadcast | tokio   | spmc    | mpmc 1P |
| --------- | --------------- | ------- | ------- | ------- |
| 1         | 109 µs          | 120 µs  | 45.1 µs | 145 µs  |
| 2         | 218 µs          | 149 µs  | 91.2 µs | 251 µs  |
| 4         | 421 µs          | 161 µs  | 112 µs  | 639 µs  |
| 8         | 913 µs          | 188 µs  | 141 µs  | 1.49 ms |
| 32        | 3.30 ms         | 609 µs  | 359 µs  | 5.45 ms |

## Reading

Native-thread bounded waits use a shared condvar (`recv_blocking` / `send_blocking`) instead of one async waker per parked task. Cap 64 / 32 receivers is `46.8 ms` vs `async-broadcast` `39.8 ms`. Cap 1 / 32 is `328.8 ms` vs 1-producer mpmc `239.5 ms`. Scheduled and round-trip cells stay ahead of the lossless peers.

Unbounded send publishes without the waiter mutex when receivers exist. Fan-out at 32 receivers is `359 µs` vs Tokio’s lossy ring at `609 µs`. Unbounded try/ready round-trip is `7.11 ns` / `10.5 ns` vs Tokio `16.8 ns` / `21.0 ns`. The leftover bounded-vs-unbounded gap is wait-at-capacity parking, not the drain path.
