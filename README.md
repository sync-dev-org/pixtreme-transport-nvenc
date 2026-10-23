# pixtreme-transport-nvenc

`pixtreme-transport-nvenc` provides a shared library with a C ABI for H.265 encoding with
NVIDIA NVENC. C and C++ output modules link the encoder, submit frames in CUDA device
memory, and receive encoded access units with their original timestamps.

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
Encoding requires an NVIDIA GPU with HEVC NVENC support and a driver
supporting CUDA 13 and NVENC API 13.0. Driver API compatibility is checked before
creating a CUDA context or NVENC session.

The build configures CUDA architectures 86, 89 and 120. The encoder itself uses
host C++ code and NVIDIA driver APIs, with no embedded CUDA kernels. Hardware
execution has been verified on compute capability 8.6; execution on 8.9 and
12.0 hardware has not been verified.

## Use from C and C++

Version 0.2.0 provides `<pixtreme/nvenc/c/encoder.h>` for C and
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

An optional version in `find_package` sets a minimum version. Before any other
encoder operation, compare `pixtreme::transport::nvenc::runtime_abi_version()`
with the public header's `pixtreme::transport::nvenc::abi_version` and stop if
they differ. The current ABI version is 3. CMake package discovery does not
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
field through `device`; its optional colour fields default to zero
when omitted.
Loan descriptors require every field through `loan_id` even on output. Other
outputs write only complete members that fit the caller's capacity. Accepted settings, SPS colour readback and low-latency switches are returned
in three independent records, so each record can grow without moving fields in
another record. The header documents the field rules, including unknown extensions and partial fields.

Create a session with `pixtreme_transport_nvenc_session_create`. Its remaining
operations are `session_borrow_input`, `session_encode`, `return_input`,
`session_flush`, `session_close`, `session_wait_closed`, `session_take`, and
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
of zero with a null buffer is a normal size query. `BUFFER_TOO_SMALL` succeeds
without changing the buffer, consuming the AU or releasing its input, and
reports a required size greater than the supplied capacity. Allocate at least
that size and retry. Another consumer can take the queried AU in between, so
each retry checks the current front and can require more capacity. Share one
waiting deadline across retries. `ACCESS_UNIT` writes exactly the actual size
and consumes the AU once. The caller owns those bytes independently of the
session. No access-unit handle needs to be retained or released.

### Session API

All public types are in `pixtreme::transport::nvenc`. Construct a `Session`
with `Settings` specifying dimensions, input format, profile, CBR or VBR,
bitrate, frame rate, GOP length, IDR period, input-slot count and CUDA device.
The input formats are NV12 (8-bit 4:2:0, Main profile) and P010 (10-bit 4:2:0,
Main10 profile). Convert other formats before passing them to the encoder.

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
output; its input stays unavailable until successful delivery. Every submitted frame yields one
access unit in input order, with its timestamp unchanged. Encoding uses no
B-frames or lookahead and does not wait for later frames to produce an access
unit. `effective_settings()` exposes accepted settings and separate SPS colour
readback.

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

## Development

Development sources are maintained separately from the published package. A
source checkout has a standalone C++ project in `cpp/` and a Python
packaging project in `python/`. Builds require Python 3.12+, a C++17 compiler,
CMake 3.28+ and the CUDA 13 toolkit. The development checkout's
`docs/development.md` contains wheel-build, test and throughput-measurement
instructions.

## License

The package uses the MIT License. The wheel's `*.dist-info/licenses/` directory
contains the project license and the MIT notice for NVIDIA's vendored NVENC
header.
