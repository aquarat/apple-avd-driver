# apple-avd-driver

Out-of-tree build of the Apple AVD (Apple silicon video decoder) V4L2 stateless
decoder driver from AsahiLinux/linux, tag `asahi-7.1.6-1`
(`drivers/media/platform/apple/avd/` at commit
`e2e1930a9595bffafad92cec2b5504525efb9cd4`), plus a series of fixes and
experiments developed on an M1 Mac mini (T8103) running Fedora Asahi 44.

The root of this repository is the module tree (`Kbuild`, `Makefile`, the
driver sources): `make` builds `apple-avd.ko` against the running kernel.

## Branches

| branch | contents |
|---|---|
| `main` (= `series`) | pristine driver + the recommended series: patches 0001-0006 and 0008 of `notes/patches/final/` |
| **`series-7.1.13`** | driver from Fedora kernel-16k **7.1.13**-402.asahi + `notes/patches/final-7.1.13/` 0001-0004 (validated on an M1 Pro, T6000); see `notes/7.1.13/README.md` |
| `readback` | `main` up to 0006 (cacheable capture buffers) |
| `dma-coherent` | experiment: force `dev->dma_coherent` (what the RFC DT patch 0007 does) |
| `coh-probe` | experiment: per-frame cache-coherency probe on top of `dma-coherent` |
| `dbg-irq`, `exp-0567`, `min-0007`, `exp-final-v1`, `exp-early-submit-v2` | intermediate bisection builds (early-submit experiment, VP_INSN_FIFO_MASK bisection) |

## notes/

`notes/` carries the working notes, patch files and test scripts from the
directory this module tree was developed in. The original layout was
`src/avd-driver/` containing `apple-avd/` (this repository's root), the
scripts, `patches/` and `readback/`; paths in `notes/README.md` and
`notes/RESULTS.md` refer to that layout. `notes/patches/final/` are the
kernel-path versions of the commits on `main` (plus the DT-only RFC 0007),
`notes/patches/ffmpeg/` the matching FFmpeg patches (Kwiboo
`v4l2-request-n8.1`). The test clips are not included; the scripts take the
clip directory from `$RESEARCH`.

## Kernel 7.1.13

`series-7.1.13` is the branch to build on kernel-16k 7.1.13. The stock 7.1.13 driver already contains 0001-0003 of
the 7.1.6 series, but has two bugs that make it unusable for long-running decodes: **every H.264 stream stops after
4096 slices** (`slice_num` is never reset: "slice_num > 4096, stream was rejected!", then EBUSY), and a per-frame
8.4 MB `kzalloc` drops frames when memory is fragmented. `notes/7.1.13/README.md` has the analysis, the port of 0008,
the validation numbers and T6000 notes (firmware, device-node ordering).
