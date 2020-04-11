---
title: Go-ethereum EVM optimization report
---

> Report about optimization work related to using uint256 in EVM in go-ethereum.

## Introduction (obvious facts)

Ethereum Virtual Machine (EVM) uses 256-bit words. Therefore every EVM implementation requires arbitrary/extended precision integer, usually provided by a software library.

Go-ethereum's EVM uses `big.Int` for that purpose. The `big.Int` is part of Go's standard library and it is classic infinite-precision integer implementation. This means integers are represented as a tuple of `(number_of_words, words[])`, where `words[]` array is dynamically allocated and can be extended when needed. This has number of theoretical disadvantages in context of EVM needs. To mentions only two:

1. Every integer object needs dynamic memory allocation.
2. After every arithmetic operation performed additional `mod 2**256` is needed because EVM requires _wrapping_ behavior of 256-bit two's complement representation.

## uint256 library

In March 2018 Martin Swende started a small side project [uint256] with the goal to provide efficient 256-bit integer implementation for EVM. This project recently reached important milestone (marked with [0.1.0][uint256 0.1.0] release) — it now provides completely independent implementation of all operations needed by EVM. This implementation is also [significantly faster than `big.Int`][uint256 benchmarks].

## Geth with uint256 — benchmark results

To estimate the impact of new 256-bit integer implementation in context of full Ethereum client, the `big.Int` has been replaced with `uint256.Int` in geth's  EVM implementation — see the PR [go-ethereum#20787] containing this change.

In March 2020, we performed a benchmarking experiment by running two geth nodes in parallel on the Ethereum mainnet. One was vanilla geth, second was geth with `uint256` modification. Both started full blockchain sync at block 5000000.

The `geth+uint256` reached the top of the blockchain at block 9761223. At this point it was 36342 blocks ahead of `geth-vanilla`. This gave **0.73%** shorter full-sync time.

It was also observed that `geth+uint256` spent **~10%** less time on _execution_ during the sync.

Finally, we compared `big.Int` vs `uint256.Int` EVM performance on purely computational workloads using [evmone's benchmark set]. Go-ethereum's EVM with `uint256` has shorter execution times by **~22–47%**. Full results below.

```
name                       old time/op  new time/op  delta
blake2b_huff/2805nulls     3.97ms ± 0%  2.86ms ± 0%  -28.01%  (p=0.000 n=9+8)
blake2b_huff/5610nulls     7.76ms ± 0%  5.53ms ± 0%  -28.78%  (p=0.000 n=10+10)
blake2b_huff/65536nulls    88.1ms ± 1%  62.0ms ± 0%  -29.64%  (p=0.000 n=10+10)
blake2b_huff/8415nulls     11.3ms ± 1%   8.1ms ± 0%  -28.65%  (p=0.000 n=10+10)
blake2b_huff/empty          210µs ± 0%   163µs ± 0%  -22.63%  (p=0.000 n=10+10)
blake2b_shifts/2805nulls   30.2ms ± 1%  21.4ms ± 0%  -29.07%  (p=0.000 n=10+10)
blake2b_shifts/5610nulls   59.7ms ± 0%  42.6ms ± 0%  -28.56%  (p=0.000 n=10+10)
blake2b_shifts/65536nulls   682ms ± 0%   488ms ± 1%  -28.52%  (p=0.000 n=8+9)
blake2b_shifts/8415nulls   88.6ms ± 1%  63.3ms ± 0%  -28.54%  (p=0.000 n=9+10)
sha1_divs/1351             8.99ms ± 1%  5.71ms ± 0%  -36.47%  (p=0.000 n=10+9)
sha1_divs/2737             17.6ms ± 1%  11.1ms ± 0%  -36.97%  (p=0.000 n=10+10)
sha1_divs/5311             34.2ms ± 1%  21.5ms ± 0%  -37.00%  (p=0.000 n=10+10)
sha1_divs/65536             416ms ± 1%   261ms ± 0%  -37.25%  (p=0.000 n=9+10)
sha1_divs/empty             452µs ± 1%   293µs ± 1%  -35.20%  (p=0.000 n=10+10)
sha1_shifts/1351           7.88ms ± 0%  4.31ms ± 0%  -45.29%  (p=0.000 n=10+9)
sha1_shifts/2737           15.3ms ± 0%   8.4ms ± 1%  -45.44%  (p=0.000 n=9+10)
sha1_shifts/5311           29.8ms ± 1%  16.3ms ± 0%  -45.32%  (p=0.000 n=9+9)
sha1_shifts/65536           364ms ± 2%   197ms ± 0%  -45.87%  (p=0.000 n=10+9)
sha1_shifts/empty           392µs ± 1%   223µs ± 0%  -43.14%  (p=0.000 n=10+10)
weierstrudel/0             1.11ms ± 1%  0.58ms ± 0%  -47.43%  (p=0.000 n=10+9)
weierstrudel/1             2.28ms ± 0%  1.28ms ± 0%  -44.01%  (p=0.000 n=8+10)
weierstrudel/14            10.7ms ± 1%   6.0ms ± 0%  -44.22%  (p=0.000 n=10+9)
weierstrudel/3             3.58ms ± 1%  2.01ms ± 0%  -43.93%  (p=0.000 n=9+9)
weierstrudel/9             7.49ms ± 1%  4.19ms ± 0%  -44.14%  (p=0.000 n=10+10)
```

[uint256]: https://github.com/holiman/uint256
[uint256 0.1.0]: https://github.com/holiman/uint256/releases/tag/v0.1.0
[uint256 benchmarks]: https://github.com/holiman/uint256/tree/v0.1.0#benchmarks
[go-ethereum#20787]: https://github.com/ethereum/go-ethereum/pull/20787
[evmone's benchmark set]: https://github.com/ethereum/evmone/tree/master/test/benchmarks