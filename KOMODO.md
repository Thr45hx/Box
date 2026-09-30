# KOMODO in Box — on-device Tensor NPU compiling, no root

This branch adds the KOMODO v12 Google Tensor compiler plugin to Box, so the app can compile models for the
Tensor NPU **on the phone, inside a normal app** — no root, SELinux enforcing.

## What changed

| File | Change |
|---|---|
| `Android/src/app/src/main/jniLibs/arm64-v8a/libLiteRtCompilerPlugin_google_tensor.so` | **Added.** KOMODO v12 plugin (1,998,696 B, md5 `d00888012c3afcb3b67bed9f5feb204e`). Upstream Box ships Qualcomm/MediaTek compiler plugins but no Google Tensor one. |
| `Android/src/app/src/main/AndroidManifest.xml` | **Added** `<uses-native-library android:name="libedgetpu_litert.so" android:required="false"/>`. |

## Why the manifest line matters

The compiler plugin calls `dlopen("/vendor/lib64/libedgetpu_litert.so")` → `EdgeTpuCompilerCompileFlatbuffer`.
An app's classloader linker namespace can only see that vendor library if the manifest declares it.
The installed Box 3.5.5 APK appears to get the entry via manifest merge from a dependency; it is declared
explicitly here so the fork does not rely on that.

This is also why the Termux CLI needed root: binaries there run in linker `namespace.default`, which cannot
open the vendor library. It was never a root-only compiler.

**Verified 2026-09-29 (Pixel 9 Pro XL, Tensor G4):** a throwaway `untrusted_app` (targetSdk 37, SELinux
enforcing, no `su`) loaded this plugin with `System.load` and compiled a one-op conv to a 19,392-byte module
containing a DGC0 — byte-identical to `apply_plugin_main.komodo_v12.arm64` output.

## Rules for using it

- **Load the plugin in-process** (`System.load` / JNI). Do **not** `exec` a compiler binary from the app's data
  dir — that falls back to `namespace.default` and loses the vendor library again.
- The plugin is an **x86-SDK-free** path: compiling happens on the device's own vendor compiler.

## Still to do

1. **Compile screen** — pick model → in-process compile → save DGC → run. Not built yet.
2. **YNNPACK tier** — not a jniLibs drop. Box's `liblitertlm_jni.so` registers XNNPACK only; the int2
   embedder/aux CPU tier needs a runtime rebuilt with the YNNPACK delegate, registered after NPU and GPU.
3. On-device test of this branch's APK with a real model (only the probe app has been tested so far).
4. **G6:** this plugin was built and validated for Tensor G4. Re-validate on G6.
