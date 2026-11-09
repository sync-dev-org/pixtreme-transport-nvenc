# pixtreme-transport-nvenc

`pixtreme-transport-nvenc` provides a shared library with a C ABI for H.264,
H.265 (HEVC) and AV1 encoding with NVIDIA NVENC. C and C++ output modules link
the encoder, write each frame into an encoder-owned CUDA region lent as a
[pixtreme](https://pypi.org/project/pixtreme/) `PixelBuffer`, and receive
encoded access units with their original timestamps.

One wheel includes the shared library, public C header and header-only C++ wrapper, CMake package
configuration, and license notices. It depends on the pixtreme wheel and
exposes no Python encoder bindings.

## Install

```bash
pip install pixtreme-transport-nvenc
```

For a uv project:

```bash
uv add pixtreme-transport-nvenc
```

The package depends on `pixtreme>=2.11.0`, whose wheel supplies the
`PixelBuffer` type and `libpixtreme.so.2`; the encoder finds that library in
the same environment. At runtime, the encoder loads `libcuda.so.1` and
`libnvidia-encode.so.1` from the host's NVIDIA driver. Neither driver library
nor the CUDA runtime is bundled in the wheel.

## Supported environments

PyPI wheels support Linux x86_64 with glibc 2.28 or later
(`manylinux_2_28_x86_64`), including WSL2, and Python 3.12 or later. One
`py3-none` wheel serves every supported Python version. PyPI releases contain
wheels only; no source distribution is uploaded. Installation on another
operating system or CPU architecture fails with no matching distribution.
Encoding requires an NVIDIA GPU with NVENC support for the selected codec and
configuration. The driver must support CUDA 13 and NVENC API 13.0 (driver
branch 580 or later). Driver API compatibility is checked before creating a
CUDA context or NVENC session. The encoder calls NVIDIA driver APIs directly;
the shared library contains host C++ code and no embedded CUDA kernels.

Supported configurations depend on runtime queries for the codec, profile,
input format, output precision and chroma format, dimensions, B frames and
lookahead. Representative verified capabilities include:

| GPU | Compute capability | Examples of verified support |
|---|---|---|
| RTX A6000 (Ampere) | 8.6 (sm 86) | H.264 8-bit; HEVC 8-bit and 10-bit |
| RTX 6000 Ada Generation | 8.9 (sm 89) | H.264 and HEVC; AV1 Main 4:2:0 at 8 or 10 bits |
| GeForce RTX 5090 (Blackwell) | 12.0 (sm 120) | H.264 10-bit; H.264 and HEVC 4:2:2 with NV16/P210; AV1 Main 4:2:0 at 8 or 10 bits |

The GPU model alone does not decide whether a requested combination is valid.
Unsupported GPU configurations are rejected before a session is created, with
an `unsupported` error reason identifying the codec and setting. Invalid
arguments and conflicting settings use `invalid_configuration`. Unsupported
configurations are also distinguished from driver-query, initialization and
resource-allocation failures.

## Use from C and C++

The package provides `<pixtreme/nvenc/c/encoder.h>` for C and
`<pixtreme/nvenc/encoder.hpp>` for C++17. The C++ wrapper is header-only;
only C functions cross the shared-library boundary. Both headers include
pixtreme's `<pixtreme/pixel_buffer.h>` and require neither CUDA headers nor
core headers. The package configuration finds the pixtreme SDK installed in the
same environment. Locate the installed CMake
prefix using the Python interpreter from the environment containing the wheel:

```bash
nvenc_prefix="$(python -c 'import sysconfig; print(sysconfig.get_path("platlib") + "/pixtreme/transport/nvenc")')"
cmake -S /path/to/consumer -B /path/to/consumer/build \
  -DCMAKE_PREFIX_PATH="$nvenc_prefix"
cmake --build /path/to/consumer/build
```

Use `.venv/bin/python` in place of `python` when the wheel is installed in a uv
virtual environment that has not been activated. The consumer's
`CMakeLists.txt` discovers the package and links its imported target:

```cmake
cmake_minimum_required(VERSION 3.28)
project(output_module LANGUAGES CXX)
find_package(pixtreme_transport_nvenc CONFIG REQUIRED)
add_executable(output_module main.cpp)
target_link_libraries(output_module PRIVATE pixtreme::transport_nvenc)
```

The target supplies the public include directories, the pixtreme target,
C++17 requirement and shared library. CMake gives the build executable a
runtime search path to the encoder and pixtreme. An installed consumer must
retain an appropriate `INSTALL_RPATH`. For a shared library installed at
`pixtreme/transport/output/liboutput.so`, use paths relative to that library:

```cmake
target_link_libraries(output PRIVATE pixtreme::transport_nvenc)
set_target_properties(output PROPERTIES INSTALL_RPATH "$ORIGIN/../nvenc;$ORIGIN/../../core/lib")
install(TARGETS output LIBRARY DESTINATION pixtreme/transport/output)
```

Keep the encoder in its own wheel. If the consumer wheel uses `auditwheel
repair`, exclude `libpixtreme_nvenc.so`, `libpixtreme.so.2` and the NVIDIA
driver libraries from copying. Relative `$ORIGIN` paths allow the Python environment to move without
breaking library loading.

Release tags use `vX.Y.Z`. CMake uses the numeric release segment of the wheel
version. Package versions and the runtime ABI version advance independently.

An optional version in `find_package` sets a minimum version. Before any other
encoder operation, compare `pixtreme::transport::nvenc::runtime_abi_version()`
with the public header's `pixtreme::transport::nvenc::abi_version` and stop if
they differ. The current ABI version is 6. CMake package discovery does not
perform this runtime check. C++ classes, exceptions and automatic resource
management stay in the consumer's header-only wrapper, so consumers can use a
different compiler and standard library from the encoder.

### C API

For a C consumer, set `project(output_module LANGUAGES C)` in the CMake
example and include `<pixtreme/nvenc/c/encoder.h>`. Before any other operation,
compare `pixtreme_transport_nvenc_runtime_abi_version()` with
`PIXTREME_TRANSPORT_NVENC_ABI_VERSION` and stop if they differ.

C types and functions use the `pixtreme_transport_nvenc_` prefix. Start each
record with `struct_size = sizeof(record)`, the record size defined by the
caller's header rather than the capacity of a larger allocation. A settings
record requires every field this header defines; fields that a later header
appends use defaults when absent. Outputs write only complete members that fit
the caller's capacity. Accepted settings and low-latency switches are returned
in two independent records, so each record can grow without moving fields in
the other record. The header documents the field rules, including unknown
extensions and partial fields.

Create a session with `pixtreme_transport_nvenc_session_create`. Its remaining
operations are `session_acquire_input`, `session_encode`, `session_flush`,
`session_close`, `session_take` and `session_effective_settings`, with the same
prefix. Input regions are `pixtreme_pixel_buffer_t` handles, and wait durations
use nonnegative `int64_t` milliseconds. The functions return an `int32_t`
status and optionally an owned error handle; read its reason, classification
and message, then release it with `pixtreme_transport_nvenc_error_release`. No
exception escapes a C function. An exhausted input pool, a full access-unit
queue, pending output and an unconfirmed close return success with a separate
result indicating that state.

A newly returned session or error owns one reference. Use the matching
retain/release functions for copies and final release. Releasing a session's
last reference starts closing it without waiting. `session_acquire_input`
returns an owning PixelBuffer, which the caller releases with pixtreme's
`pixtreme_pixel_buffer_release` or `pixtreme_pixel_buffer_destroy`. The C++
wrapper performs this reference management automatically.

For output, pass a writable host buffer and its capacity to `session_take`,
along with pointers for size, a sized
`pixtreme_transport_nvenc_access_unit_metadata` record and the take status.
The record supplies `timestamp`, `is_idr`, `display_order` and
`picture_type`. A capacity of zero with a null buffer is a normal size query.
`PIXTREME_TRANSPORT_NVENC_BUFFER_TOO_SMALL` succeeds without changing the
buffer or consuming the AU, and reports a required size greater than the
supplied capacity. Allocate at least that size and retry. Another consumer can
take the queried AU in between, so each retry checks the current front and can
require more capacity. Share one waiting deadline across retries.
`PIXTREME_TRANSPORT_NVENC_ACCESS_UNIT` writes exactly the actual size and
consumes the AU once. The caller owns those bytes, which stay valid after the
session is released. No access-unit handle needs to be retained or released.

### Session API

All public types are in `pixtreme::transport::nvenc`. Construct a `Session`
with `Settings` specifying `codec`, dimensions, `input_format`, `profile`,
rate control, frame rate, GOP length, `idr_period`, `input_slots` and CUDA
`device`. Select `Codec::h264`, `Codec::hevc` or `Codec::av1` through the same
API. The positive rational `frame_rate` is used for rate control; it does not
generate timestamps or check input intervals. `idr_period` sets the IDR period
for H.264/HEVC and the key-frame period for AV1.

`Profile::main` selects the chosen codec's Main profile. HEVC also provides
`Profile::main10` and `Profile::frext`. H.264 provides `Profile::baseline`,
`Profile::high`, `Profile::high10`, `Profile::high422`, `Profile::high444`,
`Profile::stereo`, `Profile::progressive_high` and `Profile::constrained_high`.
`Profile::autoselect` selects the runtime-listed automatic profile. Each
selection must agree with the codec, precision and chroma format.

Set `input_bit_depth`, `output_bit_depth` and `chroma_format` to select the
requested input/output precision and output sampling. Zero precision fields
and `ChromaFormat::unspecified` use the input format's defaults. The input
precision must match the selected format, and runtime support determines the
permitted output combinations.

Choose `RateControl::cbr` with a target `bitrate`, `RateControl::vbr` with
average `bitrate` and `max_bitrate`, or `RateControl::constqp` with
`qp.intra`, `qp.inter_p` and `qp.inter_b`. Lossless encoding requires
`lossless=true`, CONSTQP and zero for all three QPs, and is available only
where the codec and GPU support it. B frames are controlled by `b_frames`;
lookahead is controlled by `enable_lookahead` and `lookahead_depth`.

`input_slots` selects the fixed number of input regions. At least `1+B`
regions are required without lookahead and `1+B+L` with lookahead, where
`B=b_frames` and `L=lookahead_depth`. A smaller count is an
`invalid_configuration` error during session creation. For example, two B
frames and lookahead depth four require at least seven regions. When every
region is in use, `acquire_input()` returns without waiting and reports that
none is available. While `access_unit_queue_limit` (at least one) access units
wait for `take()`, `encode()` returns `false` without consuming its input.
Inputs that NVENC is still reading add their access units after `encode()`
accepted them, so up to `access_unit_queue_limit + input_slots - 1` access
units can wait.

#### Input formats

The API covers the SDK 13.0 formats that a pixtreme `PixelFormat` describes,
subject to codec and GPU support. Each lent PixelBuffer has the corresponding
pixtreme format.

| `InputFormat` value | pixtreme `PixelFormat` | Input precision and layout | Planes |
|---|---|---|---|
| `InputFormat::nv12` | `nv12` | 8-bit 4:2:0, Y then interleaved UV | 2 |
| `InputFormat::p010` | `p010` | 10-bit 4:2:0, Y then interleaved UV | 2 |
| `InputFormat::yv12` | `yv12` | 8-bit 4:2:0, Y then V then U | 3 |
| `InputFormat::iyuv` | `yuv420p` | 8-bit 4:2:0, Y then U then V | 3 |
| `InputFormat::yuv444` | `yuv444p` | 8-bit 4:4:4, Y then U then V | 3 |
| `InputFormat::yuv444_10bit` | `yuv444p10msb` | 10-bit 4:4:4, Y then U then V | 3 |
| `InputFormat::nv16` | `nv16` | 8-bit 4:2:2, Y then interleaved UV | 2 |
| `InputFormat::p210` | `p210` | 10-bit 4:2:2, Y then interleaved UV | 2 |
| `InputFormat::ayuv` | `vuya` | 8-bit packed V8U8Y8A8 bytes | 1 |

The packed RGB values `argb`, `abgr`, `argb10` and `abgr10` remain reserved
for the SDK's RGB inputs. pixtreme 2.11.0 has no `PixelFormat` for them, so
session creation rejects them as an `invalid_configuration` error.

#### Input and output

Call `acquire_input()` to obtain a free input region as a
`pixtreme::pixel_buffer`. It wraps encoder-owned CUDA memory in the session's
format, dimensions and NVENC plane layout, with unknown colour declarations.
Write the frame into it, for example with pixtreme's `to_pixel_buffer`, and
record the write in pixtreme's use records (pixtreme's conversions record their
writes; record other work with `pixtreme_ndarray_record_stream`). Then pass the
PixelBuffer and a `std::uint64_t` timestamp to `encode()`. NVENC waits on the
GPU for the recorded uses, so the caller needs no host synchronization. The
encoder uses the selected device's primary CUDA context, and the caller must
write on that device.

`encode()` submits work without waiting for encoding to finish and accepts
only a region lent by the same session that has not been encoded. The encoder
holds its own reference until NVENC has read the input. A region becomes free
again once every reference is released and the work on the final release
stream completes; taking its access unit is not required. Release a region that
was never encoded with the stream of its last use. Once the session has been
released and its close has finished, releasing the last region that the caller
still holds waits inside that release call until the work on the regions'
release streams completes; the wait does not keep the CPU busy. Call
`take(std::chrono::milliseconds)` to receive an access unit, a not-ready result,
or the end of the stream after flushing. The wrapper prepares its vector before
consuming an AU. If allocation fails, a later `take()` can retry the same queued
output.

Every submitted picture yields one access unit, including when identical
pixels are submitted again. Access units are returned in coding order, which
can differ from display order with B frames. Each unit retains the caller's
timestamp unchanged, including duplicate timestamps. `display_order`
identifies the input picture in display order, and `picture_type` identifies
the codec's picture category. `is_idr` identifies H.264/HEVC IDR pictures and
AV1 key pictures.

For H.264/HEVC, an AU contains one coded picture and its associated NAL units.
For AV1 without scalability, it is a temporal unit containing one shown
picture and the preceding no-show frame OBUs needed for that display, in
coding order. The display may use `show_existing_frame` to show a previously
coded reference. Its metadata and timestamp identify the shown input picture.

With B frames and lookahead disabled, an input's AU is returned after encoding
without waiting for a later input. Enabling either feature allows output to
be delayed. `effective_settings()` exposes the accepted settings, including
the colour request, and the low-latency switches.

`flush()` ends input without waiting for the remaining access units. Continue
calling `take()` to drain them. `close(timeout)` stops input and waits up to
the timeout until NVENC no longer reads any accepted input and the internal
collector has exited; it returns `false` while that is unconfirmed and can be
called again. Outstanding outputs remain available through `take()` after
close. A region the caller still holds keeps its memory until its final
release. A cleanup failure throws a fatal `EncoderError` distinct from the
original output error. Other operation failures also use `EncoderError`, which
exposes their reason and recoverable or fatal classification.

### Colour signalling

`Settings::colour` takes H.273 codes through `ColourSignalling`: `primaries`,
`transfer`, `matrix` and `full_range` (0 for limited range, 1 for full range).
Each value is specified or left absent on its own; absent codes are written as
2 (unspecified) and an absent range as 0. H.264 and HEVC use the SPS VUI: any
specified value sets `video_signal_type_present_flag=1` with `video_format=5`
(unspecified), and any specified code sets `colour_description_present_flag=1`.
With all four absent, H.264/HEVC set `video_signal_type_present_flag=0`. AV1
uses the sequence header's `color_config`, which always carries a range; it
sets `color_description_present_flag=0` when all three codes are absent or 2.

`chroma_location` supplies top and bottom field H.273 types through
`ChromaLocation`. It is valid only for 4:2:0 output; without it, H.264/HEVC set
`chroma_loc_info_present_flag=0`. H.264/HEVC accept types 0–5; progressive
content should normally use the same type for both fields. AV1 requires equal
types that map to its single `chroma_sample_position`: type 0 maps to position
1 and type 2 maps to position 2. Reserved codes, out-of-range values and colour
requests that the selected codec and chroma format cannot express are rejected
as `invalid_configuration` before session creation. For example, AV1 infers
full range for BT.709 primaries with the sRGB transfer and the identity matrix,
so that combination requires `full_range` 1. H.264 and HEVC allow the matrix
codes 0 (identity), 16 (YCgCo-Re) and 17 (YCgCo-Ro) only for 4:4:4 output, so
4:2:0 and 4:2:2 requests with those codes are rejected.

`effective_settings().accepted.colour` returns the colour request. The encoder
does not read colour back from its output; inspect the bitstream to check the
written values.

## Measured throughput

In 60-second HEVC measurements on an RTX A6000 under WSL2 (driver 616.92,
CUDA 13.2.86, Release build), both NV12/Main and P010/Main10 were measured at
1920x1080 at 30000/1001 fps, 3840x2160 at 30000/1001 fps, and 1920x1080 at
60000/1001 fps. Each configuration used four input regions, no B frames or
lookahead, CBR at 10 Mbps for 1080p or 35 Mbps for 2160p. GOP and IDR periods
were 60 frames at 30000/1001 fps and 120 frames at 60000/1001 fps. Inputs
were tested both as fast as possible and at the specified frame rate. All
submitted inputs produced an AU. Submitting as fast as possible, every
configuration encoded at least 220 fps at 1920x1080 and at least 149 fps at
3840x2160. At the specified frame rate, the paced runs skipped no inputs,
except the 1920x1080 NV12/Main run at 60000/1001 fps, which skipped one of
3,597 scheduled inputs because no input region or access unit queue space
was free at its scheduled time. The measured GPU was reserved for encoding
and the benchmark was pinned to two logical CPUs; work on a separate GPU ran
concurrently on other CPUs. These measurements cover the encoder with a CUDA
test-picture producer, excluding the core conversion and output module.

## Distribution

Versioned wheels are published on PyPI and in the public
[GitHub Releases](https://github.com/sync-dev-org/pixtreme-transport-nvenc/releases).
Each release includes the wheel and `SHA256SUMS` for integrity checks.

## License

The package uses the MIT License. The wheel's `*.dist-info/licenses/` directory
contains the project license and the MIT notice for NVIDIA's vendored NVENC
header.
