# jrottenberg-ffmpeg-sm_75_120

Custom build of [jrottenberg/ffmpeg](https://github.com/jrottenberg/ffmpeg) **9.0.2** (NVIDIA variant, `docker-images/9.0/nvidia2404`) tuned so the CUDA stack covers GPUs from **Turing (sm_75) through Blackwell (sm_120)**.

Built image: `boris271142/ffmpeg-sm_75_120` (Docker Hub).

## Differences from the original jrottenberg/ffmpeg

### 1. CUDA base image: `12.6.2` → `12.8.1`

Original pins `nvcr.io/nvidia/cuda:12.6.2-{devel,base}-ubuntu24.04`. CUDA 12.6's `nvcc` does not know `compute_120`/`sm_120` (Blackwell consumer, added in CUDA 12.8), so the base images are bumped to **12.8.1** (builder `devel`, runtime `base`).

CUDA 13.x was deliberately **not** used: it drops `sm_75` (Turing), breaking the target range.

### 2. ffmpeg `configure` gains `--nvccflags`

```
--nvccflags='-gencode arch=compute_75,code=compute_75 -O2'
```

The original build uses ffmpeg's default nvcc flags (`-gencode arch=compute_30,code=sm_30 -O2`).

**Why a single `compute_75` PTX and not the multi-arch SASS list (`sm_75,sm_80,sm_86,sm_89,sm_90,sm_120`)?**
ffmpeg's CUDA filters (`scale_cuda`, `yadif_cuda`, `overlay_cuda`, …) are compiled to a **single PTX blob** (`%.ptx: %.cu` in `ffbuild/common.mak`), gzipped and embedded into the binary, then JIT-compiled by the GPU driver at runtime. ffmpeg's `configure` hard-appends `-ptx` to `nvccflags` (configure:7206), and `nvcc` rejects `-ptx` when combined with multiple `-gencode` architectures:

```
nvcc fatal : Option '--ptx (-ptx)' is not allowed when compiling for multiple GPU architectures
```

So multi-SASS `--nvccflags` cannot work with ffmpeg as designed. PTX is **forward-compatible**: PTX built for `compute_75` JITs on every GPU from sm_75 upward — sm_75, 80, 86, 89, 90, 100, 120 — covering the whole target range with one blob.

Note: **NVENC/NVDEC/CUVID/libnpp do not use nvcc at all** (they are header-only, `ffnvcodec`), so hardware encode/decode works on all supported GPUs regardless of these flags. The nvccflags only affect the CUDA *filters*.

### 3. Build parallelism: `-j4` → `-j24`

`ENV MAKEFLAGS="-j24"` (original: `-j4`) — sized for a 24-CPU build host.

### 4. NVENC/NVDEC headers: `11.1.5.3` → `13.1.15.0`

`ENV NVIDIA_HEADERS_VERSION=13.1.15.0` (original: `11.1.5.3`) — the NVENC API 13.1 headers, matching the NVENC version loaded by modern drivers (570+/610.x load "Nvenc version 13.1").

## Build

```bash
docker build -t boris271142/ffmpeg-sm_75_120:latest .
```

## Verify

```bash
docker run --rm boris271142/ffmpeg-sm_75_120 -hide_banner -buildconf | grep nvccflags
# -> --nvccflags='-gencode arch=compute_75,code=compute_75 -O2'

docker run --rm --gpus all boris271142/ffmpeg-sm_75_120 \
  -f lavfi -i testsrc=duration=1:size=320x240:rate=30 -c:v h264_nvenc -f null -

docker run --rm --gpus all boris271142/ffmpeg-sm_75_120 \
  -f lavfi -i testsrc=duration=1:size=320x240:rate=30 \
  -vf "hwupload_cuda,scale_cuda=w=160:h=120" -f null -
```

## Multi-GPU hosts: how to select a GPU

On a host with several GPUs, restricting a container to a single GPU with
`--gpus '"device=N"'` can break NVENC/NVDEC on some cards: ffmpeg sees the GPU
via CUDA, but encoding fails with

```
[h264_nvenc] OpenEncodeSessionEx failed: unsupported device (2)
[h264_nvenc] No capable devices found
```

Observed on a 3-GPU host (2× RTX 3090 + RTX 3080 Ti): the 3080 Ti fails in a
container started with `--gpus '"device=1"'`, while the 3090s are fine — the
driver's NVENC unit mapping breaks when only one GPU is exposed and it becomes
CUDA device 0. (Not a hardware defect: the same card encodes fine on the host,
and the stock `jrottenberg/ffmpeg` image fails in the same container setup.)

**Fix: expose all GPUs, then pin the target GPU inside the container by UUID.**

```bash
# 1. find the GPU UUIDs on the host:
nvidia-smi --query-gpu=index,name,uuid --format=csv
# 1, NVIDIA GeForce RTX 3080 Ti, GPU-4fccb0bc-ead6-3306-6d15-b1d1cb6f8f9c

# 2. run with ALL GPUs exposed + CUDA_VISIBLE_DEVICES=<UUID> of the target:
docker run --rm --gpus all \
  -e CUDA_VISIBLE_DEVICES=GPU-4fccb0bc-ead6-3306-6d15-b1d1cb6f8f9c \
  -v /path/to/media:/media boris271142/ffmpeg-sm_75_120 \
  -hide_banner -y -i /media/input.mp4 \
  -c:v hevc_nvenc -preset p5 -cq 28 -c:a copy /media/output.mp4
```

Notes:

- Use the **UUID**, not an index: CUDA's enumeration order can differ from
  `nvidia-smi`'s index order, so `CUDA_VISIBLE_DEVICES=1` may select a
  different card than expected. The UUID is unambiguous.
- `CUDA_VISIBLE_DEVICES` accepts a comma-separated UUID list to allow several GPUs.
- `--gpus all` + UUID pinning also keeps NVENC working on the other GPUs, so a
  single command form works for every card on the host.

## Files

| File | Role |
|---|---|
| `Dockerfile` | patched from `docker-images/9.0/nvidia2404/Dockerfile` (CUDA 12.8.1, `-j24`, headers 13.1.15.0) |
| `build_source.sh` | patched: `--nvccflags` added to ffmpeg's `./configure` |
| `download_tarballs.sh`, `generate-source-of-truth-ffmpeg-versions.py`, `install_ffmpeg.sh` | unmodified from upstream |

Upstream: https://github.com/jrottenberg/ffmpeg (MIT-licensed parts; ffmpeg itself is LGPL/GPL with `--enable-gpl --enable-nonfree` components as in the original image).

## `ffmpeg -version` output

```
ffmpeg version 9.0.2 Copyright (c) 2000-2026 the FFmpeg developers
built with gcc 13 (Ubuntu 13.3.0-6ubuntu2~24.04)
configuration: --disable-debug --disable-doc --disable-ffplay --enable-cuda --enable-cuda-nvcc --nvccflags='-gencode arch=compute_75,code=compute_75 -O2' --enable-cuvid --enable-fontconfig --enable-gpl --enable-libaom --enable-libaribb24 --enable-libass --enable-libbluray --enable-libdav1d --enable-libfdk-aac --enable-libfontconfig --enable-libfreetype --enable-libfribidi --enable-libharfbuzz --enable-libkvazaar --enable-libmp3lame --enable-libnpp --enable-libopencore-amrnb --enable-libopencore-amrwb --enable-libopenjpeg --enable-libopus --enable-libsrt --enable-libsvtav1 --enable-libtheora --enable-libvidstab --enable-libvmaf --enable-libvorbis --enable-libvpx --enable-libwebp --enable-libx264 --enable-libx265 --enable-libxvid --enable-libzimg --enable-libzmq --enable-nonfree --enable-nvenc --enable-openssl --enable-shared --enable-small --enable-version3 --enable-whisper --extra-cflags='-I/opt/ffmpeg/include -I/opt/ffmpeg/include/ffnvcodec -I/usr/local/cuda/include/ -I/usr/include/x86_64-linux-gnu -I/usr/include/aarch64-linux-gnu' --extra-ldflags='-L/opt/ffmpeg/lib -L/usr/local/cuda/lib64 -L/usr/local/cuda/lib32/ -L/usr/lib/x86_64-linux-gnu -L/usr/lib/aarch64-linux-gnu -L/usr/lib' --extra-ldflags=-L/opt/ffmpeg/lib/aarch64-linux-gnu --extra-ldflags=-L/opt/ffmpeg/lib/x86_64-linux-gnu --extra-libs=-ldl --extra-libs=-lm --extra-libs=-lpthread --ld=g++ --prefix=/opt/ffmpeg
libavutil      61.  1.102 / 61.  1.102
libavcodec     63.  1.102 / 63.  1.102
libavformat    63.  1.102 / 63.  1.102
libavdevice    63.  1.102 / 63.  1.102
libavfilter    12.  1.102 / 12.  1.102
libswscale     10.  1.102 / 10.  1.102
libswresample   7.  1.102 /  7.  1.102

Exiting with exit code 0
```
