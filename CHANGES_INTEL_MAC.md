# Changes for Intel Mac (AMD GPU) Compatibility

## Branches

| Fork | Branch | Link |
|------|--------|------|
| `wendlinga/fmt` | `fix/intel-mac-missing-cstdlib` | https://github.com/wendlinga/fmt/tree/fix/intel-mac-missing-cstdlib |
| `wendlinga/plume` | `fix/intel-mac-amd-gpu` | https://github.com/wendlinga/plume/tree/fix/intel-mac-amd-gpu |
| `wendlinga/N64Recomp` | `fix/intel-mac-amd-gpu` | https://github.com/wendlinga/N64Recomp/tree/fix/intel-mac-amd-gpu |
| `wendlinga/N64ModernRuntime` | `fix/intel-mac-amd-gpu` | https://github.com/wendlinga/N64ModernRuntime/tree/fix/intel-mac-amd-gpu |
| `wendlinga/rt64` | `fix/intel-mac-amd-gpu` | https://github.com/wendlinga/rt64/tree/fix/intel-mac-amd-gpu |
| `wendlinga/BanjoRecomp` | `fix/intel-mac-amd-gpu` | https://github.com/wendlinga/BanjoRecomp/tree/fix/intel-mac-amd-gpu |

---

These changes were made to build and run `BanjoRecompiled` on an Intel Mac with an AMD Radeon GPU (AMDMTLBronzeDriver) running macOS 15.7.4, using Homebrew LLVM 22 and the macOS 11.1 SDK.

---

## 1. Metal Argument Buffer crash fix

**File:** [`lib/rt64/src/contrib/plume/plume_metal.cpp`](lib/rt64/src/contrib/plume/plume_metal.cpp)

On macOS 11+, all GPUs report Metal Argument Buffers Tier 2, so the `MetalDescriptorSet` constructor skips calling `setArgumentBuffer` on the encoder. However, Intel Mac AMD GPUs don't support Metal 3, so `useDirectBufferAddresses` is false and the code falls into a path that calls `argumentEncoder->setBuffer(...)` without first binding a backing buffer — crashing inside `AMDMTLBronzeDriver -[BronzeMtlIndirectArgumentBufferEncoder setBuffer:offset:atIndex:]`.

**Fix:** In `MetalDescriptorSet::setDescriptor`, in the `DataTypePointer` case, call `setArgumentBuffer` before `setBuffer` when `useDirectBufferAddresses` is false:

```cpp
// Before (crashed on Intel Mac AMD):
argumentEncoder->setBuffer(nativeBuffer, bufferDescriptor->offset, argumentIndex);

// After:
argumentBuffer.argumentEncoder->setArgumentBuffer(argumentBuffer.mtl, argumentBuffer.offset);
argumentBuffer.argumentEncoder->setBuffer(nativeBuffer, bufferDescriptor->offset, argumentIndex);
```

---

## 2. `task_id_token_t` missing from macOS 11.1 SDK

**File:** [`lib/rt64/src/contrib/plume/contrib/metal-cpp/Metal/MTLResource.hpp`](lib/rt64/src/contrib/plume/contrib/metal-cpp/Metal/MTLResource.hpp)

`task_id_token_t` was introduced in the macOS 13 SDK. Added a fallback definition for older SDKs:

```cpp
#include <mach/mach.h>

// task_id_token_t was introduced in macOS 13. Define a fallback for older SDKs.
#ifndef task_id_token_t
typedef mach_port_t task_id_token_t;
#endif
```

---

## 3. `maximumFramesPerSecond` missing from macOS 11.1 SDK

**File:** [`lib/rt64/src/contrib/plume/plume_apple.mm`](lib/rt64/src/contrib/plume/plume_apple.mm) (lines ~75, 110, 152)

`NSScreen.maximumFramesPerSecond` was introduced in macOS 12. With the 11.1 SDK, it's an undeclared method so Objective-C infers the return type as `id`. Casting `id` directly to `int` is a compile error. Fixed by going through `NSInteger` first:

```objc
// Before:
cachedRefreshRate.store((int)[screen maximumFramesPerSecond]);

// After:
cachedRefreshRate.store((int)(NSInteger)[screen maximumFramesPerSecond]);
```

---

## 4. `std::malloc` / `std::free` undeclared with Homebrew LLVM 22

**File:** [`lib/N64ModernRuntime/N64Recomp/lib/fmt/include/fmt/format.h`](lib/N64ModernRuntime/N64Recomp/lib/fmt/include/fmt/format.h)

Homebrew LLVM 22's libc++ does not implicitly include `<cstdlib>` through other headers. Added explicit include:

```cpp
#  include <cstdlib>  // std::malloc, std::free
```

---

## 5. `std::char_traits<unsigned char>` not specialized in LLVM 22

**File:** [`lib/rt64/src/render/rt64_texture_cache.cpp`](lib/rt64/src/render/rt64_texture_cache.cpp) (line ~1518)

`json::parse` was called with `std::vector<uint8_t>` iterators. Since `uint8_t` is `unsigned char`, this instantiates `std::char_traits<unsigned char>`, which is not a standard specialization and is undefined in LLVM 22's libc++. Fixed by passing `char*` raw pointers instead:

```cpp
// Before:
db = json::parse(databaseBytes.begin(), databaseBytes.end(), nullptr, true);

// After:
db = json::parse(reinterpret_cast<const char*>(databaseBytes.data()),
                 reinterpret_cast<const char*>(databaseBytes.data() + databaseBytes.size()),
                 nullptr, true);
```

---

## 6. `libz` missing for `dxc-macos` shader compiler

**File:** `lib/rt64/src/contrib/dxc/bin/lib/zlib.net/v1/lib/libz.dylib` *(created)*

`dxc-macos` has an rpath of `@executable_path/../lib` and looks for `zlib.net/v1/lib/libz.dylib`. The system `libz` is only in the dyld shared cache (no real file at `/usr/lib/libz.dylib`). Created the expected directory structure and symlinked to the Homebrew zlib:

```sh
mkdir -p lib/rt64/src/contrib/dxc/bin/lib/zlib.net/v1/lib
ln -s /usr/local/opt/zlib/lib/libz.dylib \
      lib/rt64/src/contrib/dxc/bin/lib/zlib.net/v1/lib/libz.dylib
```

---

## 7. Patches — MIPS cross-compilation type errors

All files in [`patches/`](patches/) are cross-compiled for MIPS using Homebrew Clang, which is stricter about implicit pointer type conversions that older MIPS toolchains accepted silently.

### [`patches/camera_transform_tagging.c`](patches/camera_transform_tagging.c) — line 121
`guOrthoF` expects `float (*)[4]` but `*mtx` is `Mtx *`. The EX variant stores a float matrix directly into `Mtx` memory:
```c
// Before:
guOrthoF(*mtx, ...);
// After:
guOrthoF((float (*)[4])(void *)(*mtx), ...);
```

### [`patches/culling_patches.c`](patches/culling_patches.c) — lines 164, 168
`unk20` is declared `s32` but actually stores an `AnimMtxList *`. Cast through `void *`:
```c
// Line 164 — function takes AnimMtxList **
animMtxList_setBoned((AnimMtxList **)(void *)&this->marker->unk20, ...);
// Line 168 — function takes AnimMtxList *
anim_802897D4((AnimMtxList *)(void *)&this->marker->unk20, ...);
```

### [`patches/picture_patches.c`](patches/picture_patches.c) — line 158
`&D_80368360` produces `f32 (*)[3]` but `modelRender_draw` expects `f32 *`. Remove the spurious `&`:
```c
// Before:
modelRender_draw(gfx, mtx, &D_80368360, ...);
// After:
modelRender_draw(gfx, mtx, D_80368360, ...);
```

### [`patches/save_extensions.c`](patches/save_extensions.c) — line 108
`savedata_8033CC98` expects `u8 *` but `save_data` is `SaveData *`:
```c
// Before:
eeprom_error = savedata_8033CC98(filenum, save_data);
// After:
eeprom_error = savedata_8033CC98(filenum, (u8 *)save_data);
```

### [`patches/snow_patches.c`](patches/snow_patches.c) — line 216
Same spurious `&` pattern as `picture_patches.c`:
```c
// Before:
mlMtx_apply_vec3f_restricted(&D_80381080, ...);
// After:
mlMtx_apply_vec3f_restricted(D_80381080, ...);
```

### [`patches/specific_actor_patches.c`](patches/specific_actor_patches.c) — line 201
Same spurious `&` pattern; `marker_getActorAndRotation` expects `f32[3]`:
```c
// Before:
this = marker_getActorAndRotation(marker, &sp34);
// After:
this = marker_getActorAndRotation(marker, sp34);
```

### [`patches/transform_tagging.c`](patches/transform_tagging.c) — lines 832, 835, 849, 852
Byte-offset pointer arithmetic produces `u8 *` but functions expect typed struct pointers:
```c
// Lines 832, 835 — cast to BKAnimationList *
animMtxList_setBoneless(&modelRenderAnimMtxList,
    (BKAnimationList *)((u8*)model_bin + model_bin->animation_list_offset_18));

// Lines 849, 852 — cast to BKModelUnk28List *
func_802E6BD0(
    (BKModelUnk28List *)((u8*)modelRenderModelBin + modelRenderModelBin->unk28), ...);
```

### [`patches/weather_patches.c`](patches/weather_patches.c) — line 263
`iPtr` is `struct4s *` but `modelRender_draw` expects a position as `f32[3]`. Use the first field explicitly:
```c
// Before:
modelRender_draw(gdl, mptr, iPtr, ...);
// After:
modelRender_draw(gdl, mptr, iPtr->unk0, ...);
```
