# Upstream PR measurements

> Speed over stock TSan, **>1 = faster**, 95 % interval in small print. State of 22 Oct 2026.

| Column | What it is | Upstream PR |
|---|---|---|
| **EA** | Escape analysis: capture-tracking limit + escape through local slots | [llvm#192945](https://github.com/llvm/llvm-project/pull/192945) + [llvm#169896](https://github.com/llvm/llvm-project/pull/169896) |
| **DE** | Dominance-based elimination, reads and writes | [llvm#169897](https://github.com/llvm/llvm-project/pull/169897) |
| **DE, reads only** | DE that keeps every write checked (fix for the write-after-report gap) | [llvm#169897](https://github.com/llvm/llvm-project/pull/169897) with F1 |
| **DE + EA** | DE with escape analysis on | both |

🟢 faster  ·  ⚪ no change  ·  🟡 unresolved (noise)  ·  🔴 slower  ·  🟰 identical code  ·  ⏳ running  ·  — not measured

## Results

| Program | Threads | EA | DE | DE, reads only | DE + EA |
|---|:-:|:-:|:-:|:-:|:-:|
| **Applications, -O0** | | | | | |
| SQLite | 8 | ⚪ 1.08 <sub>[0.99, 1.18]</sub>¹ | — | — | — |
| FFmpeg | 8 | 🟢 **1.004** <sub>[1.001, 1.009]</sub> | — | — | — |
| memcached | 8 | 🟡 0.98 <sub>[0.96, 1.01]</sub> | — | — | — |
| Redis | 8 | ⚪ 0.996 <sub>[0.961, 1.004]</sub> | — | — | — |
| MySQL | 8 | ⚪ 0.999 <sub>[0.996, 1.006]</sub> | — | — | — |
| **Applications, -O2** | | | | | |
| SQLite | 8 | 🟰 1.000 | — | — | — |
| FFmpeg | 8 | ⚪ 0.993 <sub>[0.976, 1.003]</sub> | — | — | — |
| memcached | 8 | 🟰 1.000 | — | — | — |
| Redis | 8 | 🟰 1.000 | ⏳ | ⏳ | ⏳ |
| MySQL | 8 | ⏳ | — | — | — |
| **Compute kernels** | | | | | |
| Rodinia hotspot | 8 | — | 🟢 **3.05** <sub>[2.61, 3.50]</sub> | 🟢 **3.21** <sub>[2.79, 3.84]</sub> | — |
| | 1 | — | 🟢 **1.10** <sub>[1.04, 1.16]</sub> | 🟢 **1.10** <sub>[1.04, 1.16]</sub> | — |
| NPB LU, class A | 8 | — | 🟢 **2.27** <sub>[2.03, 2.41]</sub> | 🟢 **2.24** <sub>[2.00, 2.53]</sub> | — |
| NPB LU, class W | 8 | — | 🟢 **2.02** <sub>[1.75, 2.20]</sub> | 🟢 **2.04** <sub>[1.83, 2.23]</sub> | — |
| NPB BT, class A | 8 | — | 🟢 **1.76** <sub>[1.75, 1.80]</sub> | 🟢 **1.47** <sub>[1.46, 1.49]</sub> | — |
| SPLASH-2x ocean_ncp | 8 | — | 🟢 **1.06** <sub>[1.05, 1.07]</sub> | 🟢 **1.07** <sub>[1.06, 1.08]</sub> | 🟰 = DE |
| SPLASH-2x water_nsquared | 8 | — | 🟡 1.09 <sub>[1.01, 1.16]</sub> | 🟡 1.07 <sub>[1.01, 1.17]</sub> | 🟰 = DE |
| | 1 | — | 🟢 **1.26** <sub>[1.26, 1.26]</sub> | 🟢 **1.26** <sub>[1.26, 1.26]</sub> | — |
| SPLASH-2x water_spatial | 8 | — | 🟡 1.10 <sub>[0.98, 1.19]</sub> | 🟡 1.08 <sub>[1.02, 1.15]</sub> | 🟰 = DE |
| | 1 | — | 🟢 **1.26** <sub>[1.26, 1.27]</sub> | 🟢 **1.26** <sub>[1.26, 1.27]</sub> | — |
| Rodinia srad | 8 | — | ⚪ 1.02 <sub>[0.92, 1.12]</sub> | ⚪ 1.00 <sub>[0.91, 1.12]</sub> | ⚪ 0.94 <sub>[0.91, 1.07]</sub> |
| | 1 | — | 🟢 **1.07** <sub>[1.07, 1.08]</sub> | 🟢 **1.04** <sub>[1.03, 1.05]</sub> | — |
| PARSEC streamcluster | 8 | — | ⚪ 0.98 <sub>[0.96, 1.00]</sub> | ⚪ 0.98 <sub>[0.96, 1.00]</sub> | 🔴 0.97 <sub>[0.94, 0.99]</sub> |
| PARSEC fluidanimate | 8 | — | ⚪ 0.99 <sub>[0.95, 1.04]</sub> | ⚪ 1.00 <sub>[0.97, 1.05]</sub> | 🟰 = DE |
| Rodinia lavaMD | 8 | — | 🟡 1.06-1.19 | 🟡 1.08-1.14 | 🟰 = DE |

¹ SQLite -O0 as a whole shows no change, but two tests do: checkpoint_starvation_1 🟢 **1.77×**, walthread1 🟢 **1.07×**.

"🟰 = DE": escape analysis removes nothing more in that program, so the code equals DE's. The DE + EA column uses the
full escape analysis of the paper, which is broader than the two EA PRs.

Not timed: 254 programs were screened by the share of checks DE removes; the rest fell below ~3 % or did not
separate from noise (NPB IS/SP/FT/CG/MG/EP, GAP, PARSEC/SPLASH-2x others, pbbs, LULESH, HPCCG, CoMD, miniFE,
XSBench, RSBench, xz, DuckDB, OpenSSL, zstd, RocksDB, LevelDB, pigz, brotli, Rodinia others).

## Takeaways

- 🟢 **DE gives 1.8-3× on OpenMP kernels with repeated reads** (hotspot, LU, BT) and +6 % on ocean_ncp; no change
  elsewhere.
- **EA is measurable only at -O0** (SQLite on some tests, FFmpeg +0.4-0.9 %); at -O2 the code is identical or the
  speed unchanged.
- With one thread the saving is 7-26 %; the larger 8-thread gains are consistent with less cross-core traffic on
  TSan shadow memory (not confirmed at cache-line level).
- **DE, reads only** costs speed only where it removes fewer checks than DE: BT (0.83× of DE) and srad at 1 thread
  (0.97×).
