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

| Bench              | async-broadcast | spmc     | mpmc 1P  |
| ------------------ | --------------- | -------- | -------- |
| `try_round_trip`   | 22.9 ns         | 36.9 ns  | 34.6 ns  |
| `ready_round_trip` | 39.3 ns         | 77.3 ns  | 39.7 ns  |

### Native threads (`concurrent`)

Lower time is better. 4096 messages.

| Shape            | async-broadcast | spmc     | mpmc 1P  |
| ---------------- | --------------- | -------- | -------- |
| cap 1 / 1 recv   | 14.9 ms         | 13.0 ms  | 15.1 ms  |
| cap 1 / 8 recv   | 121.7 ms        | 82.2 ms  | 77.5 ms  |
| cap 1 / 32 recv  | 528.4 ms        | 240.4 ms | 238.5 ms |
| cap 64 / 1 recv  | 503 µs          | 448 µs   | 505 µs   |
| cap 64 / 8 recv  | 10.1 ms         | 14.4 ms  | 14.2 ms  |
| cap 64 / 32 recv | 40.2 ms         | 178 ms   | 177 ms   |

### Tokio 4-worker (`scheduled`)

| Shape            | async-broadcast | spmc    | mpmc 1P |
| ---------------- | --------------- | ------- | ------- |
| cap 1 / 1 recv   | 1.21 ms         | 718 µs  | 892 µs  |
| cap 1 / 8 recv   | 13.5 ms         | 4.90 ms | 5.20 ms |
| cap 1 / 32 recv  | 67.3 ms         | 28.9 ms | 28.5 ms |
| cap 64 / 1 recv  | 207 µs          | 241 µs  | 230 µs  |
| cap 64 / 8 recv  | 2.13 ms         | 2.30 ms | 2.26 ms |
| cap 64 / 32 recv | 5.94 ms         | 8.00 ms | 8.01 ms |

## Unbounded (non-blocking batch)

Tokio and async-broadcast are given capacity 4096 so the batch fits without lag or wait.

### Round trip

| Bench              | async-broadcast | tokio   | spmc    | mpmc 1P |
| ------------------ | --------------- | ------- | ------- | ------- |
| `try_round_trip`   | 24.6 ns         | 17.0 ns | 33.7 ns | 32.7 ns |
| `ready_round_trip` | 24.1 ns         | 22.2 ns | 37.5 ns | 35.3 ns |

### Fan-out (publish 4096, then every subscription drains)

| Receivers | async-broadcast | tokio   | spmc    | mpmc 1P |
| --------- | --------------- | ------- | ------- | ------- |
| 1         | 110 µs          | 124 µs  | 171 µs  | 142 µs  |
| 2         | 222 µs          | 150 µs  | 255 µs  | 258 µs  |
| 4         | 429 µs          | 175 µs  | 681 µs  | 641 µs  |
| 8         | 902 µs          | 196 µs  | 1.58 ms | 1.48 ms |
| 32        | 3.31 ms         | 593 µs  | 5.89 ms | 5.42 ms |

## Reading

Against `async-broadcast` on the lossless bounded wait path, `spmc` is faster when capacity is 1 (especially on the Tokio runtime: about 1.7× at 1 receiver, 2.7× at 8, 2.3× at 32). With capacity 64 and many receivers, `async-broadcast` pulls ahead.

Against `tokio::sync::broadcast` on the unbounded non-blocking path, Tokio is faster. That channel overwrites when full; this comparison only uses it with room for the whole batch.

Against 1-producer `broadcast::mpmc`, `spmc` is in the same band. The exclusive-send API is the reason to pick `spmc`, not a general throughput win over that 1-producer MPMC path.
