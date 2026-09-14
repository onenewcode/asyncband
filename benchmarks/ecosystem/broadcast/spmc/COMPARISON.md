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
| `try_round_trip`   | 25.5 ns         | 21.4 ns | 33.9 ns |
| `ready_round_trip` | 40.0 ns         | 22.8 ns | 42.5 ns |

### Native threads (`concurrent`)

Lower time is better. 4096 messages.

| Shape            | async-broadcast | spmc     | mpmc 1P  |
| ---------------- | --------------- | -------- | -------- |
| cap 1 / 1 recv   | 14.9 ms         | 8.90 ms  | 14.9 ms  |
| cap 1 / 8 recv   | 128.7 ms        | 100.9 ms | 74.5 ms  |
| cap 1 / 32 recv  | 517.7 ms        | 340.8 ms | 242.6 ms |
| cap 64 / 1 recv  | 496 µs          | 367 µs   | 516 µs   |
| cap 64 / 8 recv  | 10.0 ms         | 8.68 ms  | 14.0 ms  |
| cap 64 / 32 recv | 39.2 ms         | 45.7 ms  | 168.5 ms |

### Tokio 4-worker (`scheduled`)

| Shape            | async-broadcast | spmc    | mpmc 1P |
| ---------------- | --------------- | ------- | ------- |
| cap 1 / 1 recv   | 1.23 ms         | 622 µs  | 871 µs  |
| cap 1 / 8 recv   | 14.3 ms         | 3.74 ms | 5.19 ms |
| cap 1 / 32 recv  | 67.4 ms         | 15.6 ms | 27.9 ms |
| cap 64 / 1 recv  | 224 µs          | 175 µs  | 270 µs  |
| cap 64 / 8 recv  | 2.15 ms         | 444 µs  | 2.09 ms |
| cap 64 / 32 recv | 6.23 ms         | 1.09 ms | 8.00 ms |

## Unbounded (non-blocking batch)

Tokio and async-broadcast are given capacity 4096 so the batch fits without lag or wait.

### Round trip

| Bench              | async-broadcast | tokio   | spmc    | mpmc 1P |
| ------------------ | --------------- | ------- | ------- | ------- |
| `try_round_trip`   | 25.0 ns         | 17.0 ns | 12.8 ns | 34.8 ns |
| `ready_round_trip` | 41.6 ns         | 21.7 ns | 15.6 ns | 36.4 ns |

### Fan-out (publish 4096, then every subscription drains)

| Receivers | async-broadcast | tokio   | spmc    | mpmc 1P |
| --------- | --------------- | ------- | ------- | ------- |
| 1         | 128 µs          | 134 µs  | 71.1 µs | 154 µs  |
| 2         | 221 µs          | 156 µs  | 116 µs  | 257 µs  |
| 4         | 443 µs          | 173 µs  | 150 µs  | 657 µs  |
| 8         | 946 µs          | 205 µs  | 273 µs  | 1.48 ms |
| 32        | 3.40 ms         | 521 µs  | 1.06 ms | 5.60 ms |

## Reading

Native-thread bounded waits use a shared condvar (`recv_blocking` / `send_blocking`) instead of one async waker per parked task. Cap 64 / 32 receivers is `45.7 ms` vs `async-broadcast` `39.2 ms`. Scheduled and round-trip cells stay ahead of the lossless peers.

Unbounded fan-out at 32 receivers is `1.06 ms` vs Tokio’s lossy ring at `521 µs` (about 2.0×). That remaining gap is per-slot `remaining` RMWs on a lossless log; Tokio does not wait for every subscription.
