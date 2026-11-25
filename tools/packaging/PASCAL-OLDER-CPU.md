# Engine builds for a Pascal card on an older CPU

This page is for a build that has two older parts at once:

- a **Quadro P6000** (Pascal, compute capability 6.1: `sm_61`), and
- a **CPU without AVX2**: Sandy / Ivy Bridge (AVX, no AVX2, FMA or F16C) or Westmere / Nehalem (SSE4.2 + POPCNT).

The ready-made `cuda12` engine package already runs on the P6000. What it cannot do is run the CPU expert kernels on a CPU
without AVX2: `strata generate` stops with "this CPU ... does not support AVX2 with FMA and F16C". The builds here replace
the AVX2 release baseline with the engine's experimental older-CPU floor and keep only `sm_61` machine code.

The workflow `.github/workflows/engine-pascal-old-cpu.yml` builds and uploads four packages through
`tools/packaging/build_cuda_linux.sh` and `tools/packaging/build_cuda_windows.py`:

```
strata-engine-<version>-linux-x86_64-cuda-cuda12-avx-quadro-p6000.tar.gz
strata-engine-<version>-linux-x86_64-cuda-cuda12-sse4.2-quadro-p6000.tar.gz
strata-engine-<version>-windows-x86_64-cuda-cuda12-avx-quadro-p6000.tar.gz
strata-engine-<version>-windows-x86_64-cuda-cuda12-sse4.2-quadro-p6000.tar.gz
```

The two floors:

| `STRATA_ISA_FLOOR` | CPU | Package suffix |
| --- | --- | --- |
| `avx` | Sandy Bridge (2011), Ivy Bridge (2012): AVX, no AVX2 / FMA / F16C | `avx` |
| `none` | Nehalem (2008), Westmere (2010): SSE4.2 + POPCNT | `sse4.2` |

A floor build runs only on a CPU that has that floor. It is not faster than the normal engine on an AVX2 CPU, and it is
not the release package: use it only where the release package refuses to start.

## Building one yourself

Linux, CUDA 12.9 on `PATH`:

```sh
STRATA_ISA_FLOOR=avx  VARIANT=cuda12 ARCHS=61 tools/packaging/build_cuda_linux.sh
STRATA_ISA_FLOOR=none VARIANT=cuda12 ARCHS=61 tools/packaging/build_cuda_linux.sh
```

Windows, Visual Studio 2022 and CUDA 12.9 installed:

```bat
set STRATA_ISA_FLOOR=avx
set ARCHS=61
py -3 tools\packaging\build_cuda_windows.py cuda12
```

`ARCHS=61` is the P6000. `-DSTRATA_EXPERIMENTAL_SM60=ON` is part of the `cuda12` variant, and it is what compiles the
Pascal / Volta paths (`STRATA_FUSE_DECODE`, `STRATA_PATTN_SM60`, `STRATA_SELECT_LONG`, the expert nibble kernels).
Without it those switches have nothing to select: `STRATA_SELECT_LONG` says so and is ignored, and the fused decode
and the sm_6x attention kernels are not in the binary.

The picture reader (`strata-vision`) is built for the same floor and the same `sm_61`, so a picture request works on the
old CPU too.

## Switches for this card

Set these in the launcher's environment (the app's or the Terminal's model config, `"env"`), before the engine starts.
They are read once at start. Leave one out to use the engine's own default; set it to `0` to turn it off.

```
STRATA_ADAPT_NOWAIT=1
STRATA_ADAPT_LAG=2
STRATA_SIDE_MIXER=2
STRATA_FUSE_WAIT=2
STRATA_B6_MMVQ_ROWS=1
STRATA_FUSE_DECODE=1
STRATA_GR_UP75=1
STRATA_DENSE_F32=1
STRATA_SELECT_LONG=1
STRATA_PATTN_SM60=1
```

`STRATA_FUSE_DECODE`, `STRATA_SELECT_LONG` and `STRATA_PATTN_SM60` exist only in the `STRATA_EXPERIMENTAL_SM60` build;
this build is that one. They are opt-in speed switches: the source checks the sm_6x attention kernels and the fused
decode kernels bit for bit against the reference, and the others change scheduling. Measure on the PC before keeping
the set.

## Limits

- Experimental: the same floor the engine's `STRATA_ISA_FLOOR` describes in [docs/SOURCE_INSTALL.md](../../docs/SOURCE_INSTALL.md) ("Older CPUs"). The
  project does not publish measurements for these builds. Record the CPU, the card, the package name and what was run.
- `sm_61` only: the package starts on a P6000 (and other compute-capability 6.1 cards) and on nothing else. A GTX 10 card
  of another capability, a P100 (`sm_60`) or a newer card needs its own build.
- The Windows package's Visual C++ runtime is taken from the build machine along with the CUDA DLLs
  (`package_engine.py`), so it does not need a separately installed redistributable.
