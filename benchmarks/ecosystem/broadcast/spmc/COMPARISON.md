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
| `try_round_trip`   | 24.5 ns         | 13.4 ns | 32.7 ns |
| `ready_round_trip` | 38.6 ns         | 17.2 ns | 44.3 ns |

### Native threads (`concurrent`)

Lower time is better. 4096 messages.

| Shape            | async-broadcast | spmc     | mpmc 1P  |
| ---------------- | --------------- | -------- | -------- |
| cap 1 / 1 recv   | 14.9 ms         | 9.05 ms  | 14.9 ms  |
| cap 1 / 8 recv   | 121.7 ms        | 71.9 ms  | 75.5 ms  |
| cap 1 / 32 recv  | 521.9 ms        | 216 ms   | 231 ms   |
| cap 64 / 1 recv  | 488 µs          | 230 µs   | 517 µs   |
| cap 64 / 8 recv  | 10.6 ms         | 21.0 ms  | 13.9 ms  |
| cap 64 / 32 recv | 39.7 ms         | 154 ms   | 175 ms   |

### Tokio 4-worker (`scheduled`)

| Shape            | async-broadcast | spmc    | mpmc 1P |
| ---------------- | --------------- | ------- | ------- |
| cap 1 / 1 recv   | 1.24 ms         | 565 µs  | 932 µs  |
| cap 1 / 8 recv   | 14.3 ms         | 3.60 ms | 5.26 ms |
| cap 1 / 32 recv  | 68.3 ms         | 14.9 ms | 29.1 ms |
| cap 64 / 1 recv  | 228 µs          | 109 µs  | 243 µs  |
| cap 64 / 8 recv  | 2.18 ms         | 409 µs  | 2.24 ms |
| cap 64 / 32 recv | 6.02 ms         | 1.09 ms | 7.84 ms |

## Unbounded (non-blocking batch)

Tokio and async-broadcast are given capacity 4096 so the batch fits without lag or wait.

### Round trip

| Bench              | async-broadcast | tokio   | spmc    | mpmc 1P |
| ------------------ | --------------- | ------- | ------- | ------- |
| `try_round_trip`   | 24.1 ns         | 16.6 ns | 12.7 ns | 33.4 ns |
| `ready_round_trip` | 25.7 ns         | 21.7 ns | 16.1 ns | 35.3 ns |

### Fan-out (publish 4096, then every subscription drains)

| Receivers | async-broadcast | tokio   | spmc    | mpmc 1P |
| --------- | --------------- | ------- | ------- | ------- |
| 1         | 125 µs          | 124 µs  | 74.7 µs | 169 µs  |
| 2         | 220 µs          | 151 µs  | 123 µs  | 249 µs  |
| 4         | 393 µs          | 163 µs  | 144 µs  | 608 µs  |
| 8         | 894 µs          | 200 µs  | 302 µs  | 1.37 ms |
| 32        | 3.21 ms         | 554 µs  | 949 µs  | 5.00 ms |

## Reading

Slots now hold `T` directly. Round trips and unbounded fan-out at 1–4 receivers beat every lossless peer and Tokio. Scheduled bounded runs also lead, including cap 64 / 32 receivers (`1.09 ms` vs `async-broadcast` `6.02 ms`).

Native-thread cap 64 with 8 and 32 receivers still trails `async-broadcast` (`21.0 ms` / `154 ms` vs `10.6 ms` / `39.7 ms`). Unbounded fan-out at 8 and 32 receivers is within about 1.5–1.7× of Tokio’s lossy ring (`302 µs` / `949 µs` vs `200 µs` / `554 µs`). Those cells are the remaining 1.5× misses: send still takes the waiter mutex, and many OS threads RMW the same slot `remaining` count.
