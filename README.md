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
