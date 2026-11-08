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

## Files

| File | Role |
|---|---|
| `Dockerfile` | patched from `docker-images/9.0/nvidia2404/Dockerfile` (CUDA 12.8.1, `-j24`) |
| `build_source.sh` | patched: `--nvccflags` added to ffmpeg's `./configure` |
| `download_tarballs.sh`, `generate-source-of-truth-ffmpeg-versions.py`, `install_ffmpeg.sh` | unmodified from upstream |

Upstream: https://github.com/jrottenberg/ffmpeg (MIT-licensed parts; ffmpeg itself is LGPL/GPL with `--enable-gpl --enable-nonfree` components as in the original image).
