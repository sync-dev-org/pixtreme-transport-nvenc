# pixtreme-transport-nvenc

`pixtreme-transport-nvenc` provides a shared library with a C ABI for H.264,
H.265 (HEVC) and AV1 encoding with NVIDIA NVENC. C and C++ output modules link
the encoder, submit frames in CUDA device memory, and receive encoded access
units with their original timestamps.

One wheel includes the shared library, public C header and header-only C++ wrapper, CMake package
configuration, and license notices. The package has no Python runtime
dependencies and exposes no Python encoder bindings.

## Install

```bash
pip install pixtreme-transport-nvenc
```

For a uv project:

```bash
uv add pixtreme-transport-nvenc
```

The package has no Python dependencies. At runtime, the encoder loads
`libcuda.so.1` and `libnvidia-encode.so.1` from the host's NVIDIA driver.
Neither driver library nor the CUDA runtime is bundled in the wheel.

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
only C functions cross the shared-library boundary. Both headers require
neither CUDA headers nor core headers. Locate the installed CMake
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

The target supplies the public include directory, C++17 requirement and shared
library. CMake gives the build executable a runtime search path to the encoder.
An installed consumer must retain an appropriate `INSTALL_RPATH`. For a shared
library installed at `pixtreme/transport/output/liboutput.so`, use a path
relative to that library:

```cmake
target_link_libraries(output PRIVATE pixtreme::transport_nvenc)
set_target_properties(output PROPERTIES INSTALL_RPATH "$ORIGIN/../nvenc")
install(TARGETS output LIBRARY DESTINATION pixtreme/transport/output)
```

Keep the encoder in its own wheel. If the consumer wheel uses `auditwheel
repair`, exclude `libpixtreme_nvenc.so` and the NVIDIA driver libraries from
copying. Relative `$ORIGIN` paths allow the Python environment to move without
breaking library loading.

Release tags use `vX.Y.Z`. CMake uses the numeric release segment of the wheel
version. Package versions and the runtime ABI version advance independently.

An optional version in `find_package` sets a minimum version. Before any other
encoder operation, compare `pixtreme::transport::nvenc::runtime_abi_version()`
with the public header's `pixtreme::transport::nvenc::abi_version` and stop if
they differ. The current ABI version is 4. CMake package discovery does not
perform this runtime check. C++ classes, exceptions and automatic resource
management stay in the consumer's header-only wrapper, so consumers can use a
different compiler and standard library from the encoder.

### C API

For a C consumer, set `project(output_module LANGUAGES C)` in the CMake
example and include `<pixtreme/nvenc/c/encoder.h>`. Before any other operation,
compare `pixtreme_transport_nvenc_runtime_abi_version()` with
`PIXTREME_TRANSPORT_NVENC_ABI_VERSION` and stop if they differ.

C types and functions use the `pixtreme_transport_nvenc_` prefix. Start each
record with `struct_size = sizeof(record)`. A settings record requires every
field through `device`; absent optional colour and control fields use zero
defaults. Set `struct_size` to the record size defined by the caller's header,
rather than the capacity of a larger allocation.
Loan descriptors require every field through `offset_2` for every input format,
even on output. A shorter descriptor is rejected before a loan is issued. Other
outputs write only complete members that fit the caller's capacity. Accepted settings, bitstream colour readback and low-latency switches are returned
in three independent records, so each record can grow without moving fields in
another record. The header documents the field rules, including unknown extensions and partial fields.

Create a session with `pixtreme_transport_nvenc_session_create`. Its remaining
operations are `session_borrow_input`, `session_encode`, `return_input`,
`session_flush`, `session_close`, `session_wait_closed`, `session_take`,
`session_take_with_metadata`, and
`session_effective_settings`, with the same prefix. Event handles use
`uintptr_t`, and wait durations use nonnegative `int64_t` milliseconds.
The functions return an `int32_t` status and optionally an owned error handle;
read its reason, classification and message, then release it with
`pixtreme_transport_nvenc_error_release`. No exception escapes a C function.
Exhausted inputs, pending output and a close wait timeout return success with
a separate result indicating that state.

A newly returned session, input owner or error owns one reference.
Use the matching retain/release functions for copies and final release. Each
borrowed descriptor owns its `owner` reference: encode or return consumes the
loan once, and the caller then releases that reference. Copying a descriptor
requires retaining its owner. Releasing a session closes it; an outstanding
loan remains returnable through its owner after that release.
The C++ wrapper performs this reference management automatically.

For output, pass a writable host buffer and its capacity to `session_take`,
along with pointers for size, timestamp, IDR state and take status. A capacity
of zero with a null buffer is a normal size query.
`PIXTREME_TRANSPORT_NVENC_BUFFER_TOO_SMALL` succeeds
without changing the buffer, consuming the AU or releasing its input, and
reports a required size greater than the supplied capacity. Allocate at least
that size and retry. Another consumer can take the queried AU in between, so
each retry checks the current front and can require more capacity. Share one
waiting deadline across retries. `PIXTREME_TRANSPORT_NVENC_ACCESS_UNIT`
writes exactly the actual size
and consumes the AU once. The caller owns those bytes independently of the
session. No access-unit handle needs to be retained or released.

`session_take_with_metadata` follows the same rules and receives a sized
`pixtreme_transport_nvenc_access_unit_metadata` record instead of separate
metadata scalars. The record supplies `timestamp`, `is_idr`, `display_order`
and `picture_type`. Both take operations are available. The input descriptor
stores its first two planes in the `row_stride` and `offset` arrays and its
third plane in `row_stride_2` and `offset_2`.

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
region is in use, `borrow_input()` returns without waiting and reports that
none is available.

#### Input formats

The API covers these thirteen SDK 13.0 formats, subject to codec and GPU
support. RGB inputs use NVENC's internal RGB-to-YUV conversion.

| `InputFormat` value | Input precision and layout | Planes |
|---|---|---|
| `InputFormat::nv12` | 8-bit 4:2:0, Y then interleaved UV | 2 |
| `InputFormat::p010` | 10-bit 4:2:0, Y then interleaved UV | 2 |
| `InputFormat::yv12` | 8-bit 4:2:0, Y then V then U | 3 |
| `InputFormat::iyuv` | 8-bit 4:2:0, Y then U then V | 3 |
| `InputFormat::yuv444` | 8-bit 4:4:4, Y then U then V | 3 |
| `InputFormat::yuv444_10bit` | 10-bit 4:4:4, Y then U then V | 3 |
| `InputFormat::nv16` | 8-bit 4:2:2, Y then interleaved UV | 2 |
| `InputFormat::p210` | 10-bit 4:2:2, Y then interleaved UV | 2 |
| `InputFormat::argb` | 8-bit packed A8R8G8B8 word | 1 |
| `InputFormat::abgr` | 8-bit packed A8B8G8R8 word | 1 |
| `InputFormat::argb10` | 10-bit packed A2R10G10B10 word | 1 |
| `InputFormat::abgr10` | 10-bit packed A2B10G10R10 word | 1 |
| `InputFormat::ayuv` | 8-bit packed A8Y8U8V8 word | 1 |

#### Input and output

Call `borrow_input()` to obtain an available `InputBuffer`. It describes the
encoder-owned CUDA allocation and each plane's row stride and offset. Write
the frame into that allocation, then pass the buffer, the producer-completion
CUDA event as `std::uintptr_t`, and a `std::uint64_t` timestamp to `encode()`.
The encoder uses the selected device's primary CUDA context. The caller and
its producer must use that same context.

`encode()` submits work without waiting for encoding to finish. Call
`take(std::chrono::milliseconds)` to receive an access unit, a not-ready result,
or the end of the stream after flushing. The wrapper prepares its vector before
consuming an AU. If allocation fails, a later `take()` can retry the same queued
output; its input stays unavailable until successful delivery.

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
be delayed. `effective_settings()` exposes the accepted settings and separate
bitstream colour readback.

`Session::return_input()` returns an unused loan with its producer-completion
event; omit the event only when no GPU work used the loan. `flush()` ends input
without waiting for the remaining access units. Continue calling `take()` to
drain them. `close()` stops input immediately; outstanding outputs remain
available, and an internal collector can finish work after `close()` returns.
After returning unused loans, call `wait_closed(timeout)` to confirm that all
input reads, cleanup and the collector thread have finished. A timeout returns
`false`; failed cleanup throws `EncoderError`. Other operation failures also
use `EncoderError`, which exposes their reason and recoverable or fatal
classification.

### Colour signalling

`Settings::colour` takes H.273 codes through `ColourSignalling`: `primaries`,
`transfer`, `matrix` and `full_range` (0 for limited range, 1 for full range).
Specify all four values or leave all four absent. H.264 and HEVC use the SPS
VUI; AV1 uses the sequence header's `color_config`. With all four absent,
H.264/HEVC set `video_signal_type_present_flag=0`. AV1 sets
`color_description_present_flag=0`, omits the three codes, and writes
`color_range=0`.

`chroma_location` supplies top and bottom field H.273 types through
`ChromaLocation`. It is valid only for 4:2:0 output. H.264/HEVC accept types
0–5; progressive content should normally use the same type for both fields.
AV1 requires equal types that map to its single `chroma_sample_position`:
type 0 maps to position 1 and type 2 maps to position 2. Reserved codes,
partial four-value requests and colour requests that the selected codec and
chroma format cannot express are rejected as `invalid_configuration` before
session creation.

The result of `effective_settings()` separates the colour request in
`accepted.colour` from the syntax readback in `sps_colour`, for every codec
including AV1. Fields absent from the bitstream stay absent in the readback.
AV1's written range is read back even when the four values were not specified.

## Measured throughput

In 60-second HEVC measurements on an RTX A6000 under WSL2 (driver 616.92,
CUDA 13.2.86, Release build), both NV12/Main and P010/Main10 sustained
1920x1080 at 30000/1001 fps, 3840x2160 at 30000/1001 fps, and 1920x1080 at
60000/1001 fps. Each configuration used four input regions, no B frames or
lookahead, CBR at 10 Mbps for 1080p or 35 Mbps for 2160p. GOP and IDR periods
were 60 frames at 30000/1001 fps and 120 frames at 60000/1001 fps. Inputs
were tested both as fast as possible and at the specified frame rate. All
inputs produced an AU, with no skipped inputs in the paced runs. The measured
GPU was reserved for encoding; work on a separate GPU ran concurrently. These
measurements cover the encoder with a CUDA test-picture producer, excluding
the core conversion and output module.

## Distribution

Versioned wheels are published on PyPI and in the public
[GitHub Releases](https://github.com/sync-dev-org/pixtreme-transport-nvenc/releases).
Each release includes the wheel and `SHA256SUMS` for integrity checks.

## License

The package uses the MIT License. The wheel's `*.dist-info/licenses/` directory
contains the project license and the MIT notice for NVIDIA's vendored NVENC
header.
