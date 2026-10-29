# UMBP calibration draft

This follow-up draft preserves the CPU calibration foundation separately from the
[UMBP simulator](umbp-simulation.md). It currently measures and fits **CPU→CPU only**
using the real `LocalCopyEngine`; GPU and cross-node calibration are unfinished.

Generated `profile.json` contains only `cpu_cpu`. It can be loaded by the simulator,
but a GPU or remote transfer requiring an absent path is rejected. There is no automatic
profile merge step yet. Use the simulator's complete example estimates for deployments
that need GPU or cross-node paths; those values are not calibrated measurements.

## CPU usage

The packaged sampler fits **CPU→CPU only**. Use a Release build, NumPy, Linux `taskset`,
a new output directory and available physical CPUs representative of deployment:

```bash
cmake --build build --target umbp_sim_probe -j 8
python examples/umbp/calibrate.py \
  --probe build/src/umbp/umbp_sim_probe --output /tmp/umbp-calibration \
  --cpus 4,5,6,7 --threads 1 2 4 --cache cold \
  --samples 24 --rounds 3 --max-mib 256 --concurrency 1 2 4
```

| Option | Meaning and defaults |
| --- | --- |
| `--cpus`, `--threads` | Threads are 1/2/4/8, including 1; defaults use up to eight allowed CPUs. Each N-thread run sets both `UMBP_DRAM_COPY_THREADS` and `UMBP_DRAM_WRITE_THREADS` to N. |
| `--concurrency` | 1/2/4 callers, default all three, including 1. Use `1` to skip concurrent validation. |
| `--max-mib` | Aggregate payload across all callers, default 256 MiB; destinations and storage pools need additional memory. |
| `--samples`, `--rounds` | Samples per workload and repeated rounds; defaults 24 and 3. |
| `--train-block-bytes`, `--train-batches` | Positive integer training lists, e.g. `4096 65536 1048576` and `1 8 32`. |
| `--validation-block-bytes`, `--validation-batches` | Independent validation lists, e.g. `8192 131072 2097152` and `1 4 16`. |
| `--cache` | `cold` (default, x86 source flush outside timers) or `reuse`. |

| Default workload | Block sizes | Batch counts | Uncapped grid maximum |
| --- | --- | --- | --- |
| Training | 512 B–4 MiB | 1/8/32/64/128 | 512 MiB |
| Single-caller validation | Independent sizes, 1 KiB–8 MiB | 1/4/16/128 | 1 GiB |
| Concurrent validation | 8 KiB, 512 KiB, 2 MiB | 1/16/128 | Subject to the aggregate cap |

Custom lists sample their full block/batch product, including concurrent validation;
omitted lists retain defaults and duplicates are sampled once. Training and validation
must not share a sampled `(block_bytes, batch)` pair. Overlap, empty capped grids or
insufficient independent training points fail without publishing a profile. Increase the
cap/lists to cover the working set; copy blocks do not represent full layer/ranged geometry.

Callers share one client/engine with disjoint keys/buffers and start together; thread
creation is outside timers. Runs use the first `min(N * callers, listed CPUs)` cores,
which can oversubscribe a short CPU list. Reads and writes are tested separately;
independent ranks/processes and simultaneous read/write mixes are outside this sweep.
Keep copy settings and CPU/NUMA placement consistent when reusing a profile.

`cold` flushes sources before transfer/put; get reads the just-written store. `reuse`
repeats buffers and caps default training at 8 MiB; explicit lists use `--max-mib`.
Neither mode guarantees cache hits; a reused-source fit may underestimate store reads,
so inspect API validation too.

Only real single-caller `LocalCopyEngine` Submit+Wait training fits coefficients.
Independent validation compares transfers and `BatchPut`/`BatchGet`; API timings include
trace output to `/dev/null` and exclude Python. `profile.json` loads via `UMBP_SIM_PROFILE`.
`report.json` records sampling parameters, profile hash, raw measurements and
per-concurrency median/signed errors. Tolerance is `max(2 µs, 10% of real median)`;
high error is reported, while operation/fit failures fail the run.

## Required before the calibration PR is ready

- Add real `HbmCopyEngine` and `MoriIoEngine` adapters, covering `cpu_gpu`, `gpu_cpu`,
  `gpu_gpu`, `remote_read` and `remote_write` alongside `LocalCopyEngine`'s `cpu_cpu`.
- Measure direct and staged transfers separately, preserving copy geometry and registration
  behavior; include staging costs once and keep them distinct from control/API overhead.
- Merge per-path measurements into a reusable complete profile. Preserve unmeasured entries
  explicitly as estimates and identify their source; do not silently call them calibrated.
- Validate held-out workloads against the actual local/GPU/RDMA engines, then separately
  report framework-facing API errors, concurrency and capacity assumptions.

The current draft has no automatic GPU or RDMA sampler and does not establish their
accuracy. Platform, CPU/NUMA placement, cache state and workload coverage affect fitted
values; concurrent clients/ranks and staging can require additional modeling.

Calibration math and CLI tests are in `tests/python/umbp/test_sim_calibration.py`.
Run `python -m pytest -q tests/python/umbp/test_sim_calibration.py`; these tests need NumPy,
but no UMBP build or GPU. They check fitting and validation isolation, not real-engine
performance accuracy.
