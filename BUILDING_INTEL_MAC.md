# Building on Intel Mac (AMD GPU)

This documents the changes required to build and run `BanjoRecompiled` on an
Intel Mac with an AMD Radeon GPU. By default the app crashes at launch on this
hardware due to a bug in how Metal argument buffers are set up.

Tested on: Intel MacBook Pro, AMD Radeon 555X (AMDMTLBronzeDriver), macOS 15.7.4.

---

## Toolchain

| Tool | Version | Use |
|------|---------|-----|
| Apple Clang (Xcode) | 16.0 | Host C++/ObjC/Metal compilation |
| Homebrew LLVM | 22 | MIPS cross-compilation for `patches/` only |
| Homebrew lld | 22 | MIPS linker for `patches/` only |
| Homebrew zlib | any | Runtime dependency for `dxc-macos` |

Apple Clang does not include a MIPS backend, so Homebrew LLVM remains
necessary for the `patches/` directory. It is not used for any host code.

---

## One-time setup

### 1. Install Xcode

Full Xcode (not just Command Line Tools) is required for the `metal` shader
compiler.

```sh
# Install via xcodes (recommended) or from the App Store
brew install --cask xcodes
xcodes install  # pick the latest stable release
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
```

### 2. Install Homebrew tools

```sh
brew install llvm lld zlib cmake ninja
brew install rust  # for bk_rom_decompress
```

### 3. Create the dxc libz symlink

`dxc-macos` (the shader compiler bundled with rt64) looks for `libz.dylib` via
its rpath. macOS only ships libz in the dyld shared cache — there is no
standalone file. Create the symlink once after cloning:

```sh
mkdir -p lib/rt64/src/contrib/dxc/bin/lib/zlib.net/v1/lib
ln -s /usr/local/opt/zlib/lib/libz.dylib \
      lib/rt64/src/contrib/dxc/bin/lib/zlib.net/v1/lib/libz.dylib
```

---

## Configure and build

```sh
# Create the resources directory (required before first build — see note below)
mkdir -p build/resources

cmake -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DPATCHES_C_COMPILER=/usr/local/opt/llvm/bin/clang \
  -DPATCHES_LD=/usr/local/opt/lld/bin/ld.lld

cmake --build build --target BanjoRecompiled -j$(sysctl -n hw.ncpu)
```

No `CMAKE_C_COMPILER` or `CMAKE_CXX_COMPILER` flags are needed — CMake picks
up Apple Clang from Xcode automatically.

> **Note on `mkdir -p build/resources`:** `apple_bundle.cmake` runs
> `iconutil -o resources/AppIcon.icns` without first creating the `resources/`
> directory. The directory must exist before the first build or iconutil will
> fail with "Failed to generate ICNS."

The finished app bundle will be at `build/BanjoRecompiled.app`.

---

## Source changes on this branch

### 1. Metal crash fix — `lib/rt64/src/contrib/plume/plume_metal.cpp`

**Root cause:** On macOS 11+, all GPUs are reported as Metal Argument Buffers
Tier 2. The `MetalDescriptorSet` constructor therefore skips calling
`setArgumentBuffer()` on the argument encoder (because `useArgumentBuffersTier2
= true`). However, Intel Mac AMD GPUs do not support Metal3 / direct buffer
addresses, so `useDirectBufferAddresses` is false and execution falls into a
path that calls `argumentEncoder->setBuffer(...)` without ever binding a
backing buffer — crashing inside
`AMDMTLBronzeDriver -[BronzeMtlIndirectArgumentBufferEncoder setBuffer:offset:atIndex:]`.

**Fix:** In `MetalDescriptorSet::setDescriptor`, call `setArgumentBuffer()`
before `setBuffer()` in the non-direct-address path:

```cpp
// In the else-branch of `if (device->useDirectBufferAddresses)`
argumentBuffer.argumentEncoder->setArgumentBuffer(argumentBuffer.mtl,
                                                  argumentBuffer.offset);
argumentBuffer.argumentEncoder->setBuffer(nativeBuffer,
                                          bufferDescriptor->offset,
                                          argumentIndex);
```

This call is a no-op on Apple Silicon and any other Metal3-capable hardware
since those devices take the direct-address path.

**Upstream PR:** [renderbag/plume#94](https://github.com/renderbag/plume/pull/94)

---

### 2. Patches Makefile — `patches/Makefile`

**Root cause:** Clang 21+ promotes `-Wincompatible-pointer-types` from a
warning to a hard error in C mode. The `patches/` directory is
cross-compiled for MIPS and several files contain implicit pointer type
conversions (e.g. `Mtx *` passed where `float (*)[4]` is expected) that older
compilers accepted silently.

**Fix:** Add `-Wno-incompatible-pointer-types` to `CFLAGS` in
`patches/Makefile`. This is consistent with the existing
`-Wno-cast-function-type-mismatch` flag already present for the same reason.

```diff
-    -Wall -Wextra ... -Wno-cast-function-type-mismatch -Werror=pointer-bool-conversion
+    -Wall -Wextra ... -Wno-cast-function-type-mismatch -Wno-incompatible-pointer-types -Werror=pointer-bool-conversion
```

Note: this flag is only needed with LLVM 21+. With LLVM ≤ 20 or Apple Clang
(if it ever gains a MIPS backend), the original code would compile as warnings
only.

---

## What was investigated but not needed

The following issues arose when building with Homebrew LLVM 22 + the macOS
11.1 SDK (an older approach). They are **not needed** with Apple Clang +
macOS 15 SDK:

- **`task_id_token_t` fallback** (`MTLResource.hpp`) — introduced in macOS 13
  SDK; present in macOS 15 SDK.
- **`maximumFramesPerSecond` cast** (`plume_apple.mm`) — method undeclared in
  SDK 11.1 causing return type inference as `id`; properly declared in
  SDK 15.
- **`fmt/format.h` missing `<cstdlib>`** — LLVM 22's libc++ is stricter than
  Apple's libc++ about implicit inclusion.
- **`rt64_texture_cache.cpp` json iterator cast** — same LLVM 22 libc++
  strictness issue with `std::char_traits<unsigned char>`.
