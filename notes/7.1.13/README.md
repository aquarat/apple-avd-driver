# Kernel 7.1.13 (Fedora kernel-16k 7.1.13-402.asahi) on an M1 Pro (T6000)

Branch **`series-7.1.13`** = the driver as shipped in `kernel-7.1.13-402.asahi.fc44` (extracted from the source
RPM's `patch-7.1-redhat.patch`) + the four patches in `notes/patches/final-7.1.13/`. Validated on an Apple M1 Pro
(MacBookPro18,3, T6000, AVD firmware `apple/avd-fw-v3-t0.bin` built from AsahiLinux/avd-fw), Fedora Asahi Remix 44.
Build and install exactly as for 7.1.6 (`make`, or `notes/scripts/build-and-install.sh` with the branch changed):
the module goes to `/lib/modules/$(uname -r)/updates/` and must be rebuilt for every kernel update.

## What changed relative to the 7.1.6 series

| 7.1.6 patch | on 7.1.13 |
|---|---|
| 0001 transform_8x8_mode_flag | upstream (8e8d72a673ab, 7.1.8), dropped |
| 0002 P-slice default weights | upstream (3cba4f1d5b86, 7.1.12), dropped |
| 0003 T8103 VP_INSN_FIFO_MASK | upstream (ca9a850f237f, 7.1.12), dropped |
| 0004 held-slice parse poll timeout | obsolete: 7.1.13 has a dedicated submit step and no per-slice parse poll |
| 0005 VP status dump | dropped (diagnostic only) |
| 0006 cacheable (non-coherent) MMAP buffers | **0001**, applies unchanged |
| 0008 reject slices with invalid DPB references | **0002**, ported: the check must run after `avd_run_postamble()` |
| - | **0003 new**: allocate the job segment table with `kvcalloc` |
| - | **0004 new**: reset `slice_num` when a frame is done; complete the request on the early returns |

## Bugs in the stock 7.1.13 driver (fixed by 0003 and 0004)

1. **Every H.264 stream stops after 4096 slices.** `avd_h264_run()` appends each slice to `h264_ctx->slices[]` and
   increments `slice_num`; `avd_h264_done()` frees the slices but never resets the count, which is zeroed only when
   the context is created. After 4096 slices every frame is refused with `slice_num > 4096, stream was rejected!`,
   and since that early return skips `avd_run_postamble()` the request is never completed, so every later request
   fails with `EBUSY` for the life of the stream. Measured on the unmodified module: a live 15 fps single-slice IP
   camera stream decoded normally to frame 4083 (~290 s), then stopped permanently (`speed` falling from 1.06x
   towards 0, `Device or resource busy` on every packet). Any long-running decode (an NVR) hits this within minutes.
2. **Frames are dropped under memory fragmentation.** `avd_init_job()` allocates `MAX_SLICES` (4096 for H.264)
   `struct avd_segment` entries of 2056 bytes each (8.4 MB, an order-10 allocation on a 16K-page kernel) with
   `kzalloc()` for every frame. With the page cache full, four parallel 2304x1296 decodes (8 runs x 4 outputs) gave
   **2/32** outputs identical to the single-stream reference on the stock module (`page allocation failure: order:10`
   in `avd_init_job`) and **32/32** with 0003 (`kvcalloc`; the table is CPU-only and written to MMIO with `writel()`).

## Porting note for 0002 (0008 on 7.1.6)

`avd_run_preamble()` applies the request's controls (`v4l2_ctrl_request_setup`) and `avd_run_postamble()` completes
the request (`v4l2_ctrl_request_complete`). A first port that rejected the slice before the postamble (and before
the slice copy) left the request pending: after the single rejection that happens when a client joins a live
stream mid-GOP, every following request on the context failed with `EBUSY` and a live stream decoded at 0.1x.
The check now runs directly after `avd_run_postamble()`; the slice copied just before is freed by
`avd_h264_done()`, like upstream's other error returns at that point.

## Validation (M1 Pro, final module)

| test | result |
|---|---|
| clip matrix (`notes/scripts/test_clips.sh`, 25 clips, GStreamer v4l2slh264dec vs software md5) | 25/25 bit-identical (stock 7.1.13: also 25/25 on clips; the 7.1.6 driver needed patches for 12 of them) |
| fragmentation stress (8 x 4 parallel 1296p decodes, page cache full, reader churning) | 32/32 correct, 0 allocation failures (stock: 2/32) |
| 5 live H.264 RTSP streams in parallel, 600 s, v4l2request | real time on all five (8974-11989 frames each, 0 EBUSY, no kernel messages) (stock: all stop after ~4096 slices) |
| join a live stream mid-GOP | 1.17x, 0 EBUSY (first 7.1.13 port of 0008: 0.10x) |
| ffmpeg v4l2request + scale_vulkan (Frigate detect chain), single and 4 parallel | output md5 identical to the T8103 runs |
| end to end: Frigate 0.18, 5 cameras x 5 fps detect + record, 10 min | 25.3 fps, 0 skipped, 0 ffmpeg restarts, no kernel messages |

## Operational notes (T6000 laptop)

* The in-tree probe fails without the firmware file: `Direct firmware load for apple/avd-fw-v3-t0.bin failed`.
* The AVD and the built-in camera ISP (`apple-isp`) probe in either order, so the decoder is `video0/media0` on some
  boots and `video1/media1` on others. ffmpeg's v4l2request finds the decoder through sysfs, so inside a container the
  device nodes must keep their host names: mapping `/dev/video1:/dev/video0` silently falls back to software decode.
  Pass all four nodes (`video0`, `video1`, `media0`, `media1`) or blacklist `apple_isp`.
* `test_clips.sh` assumes `/dev/video0` and `/dev/media0`; on such a machine run it with the right nodes.
