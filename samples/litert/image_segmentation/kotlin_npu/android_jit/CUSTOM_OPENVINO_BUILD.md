# Build and Bundle Custom OpenVINO for Intel NPU

This app loads Intel LiteRT plugins and their OpenVINO dependencies from the
APK. It no longer searches `/vendor/lib64` for the Intel compiler or dispatch
libraries.

## What is built

LiteRT and OpenVINO are not normally merged into one library. The deployment
contains these layers:

1. LiteRT Kotlin/runtime AAR (`com.google.ai.edge.litert:litert:2.1.6` by default).
2. `libLiteRtCompilerPlugin_IntelOpenvino.so`, built against one OpenVINO SDK.
3. `libLiteRtDispatch_IntelOpenvino.so`, built against the same SDK.
4. The matching Android OpenVINO runtime, TFLite frontend, Intel NPU plugin and
   compiler, plus their non-system dependencies.
5. The ALOS Intel NPU driver and firmware. These remain OS components.

The two LiteRT plugins and all OpenVINO libraries must be Android x86_64
binaries. Linux x86_64 libraries cannot run in an Android application.

## Where to build

Build in Linux. On this Windows machine, use WSL2 with Ubuntu, preferably with
the repository cloned into the WSL filesystem such as `~/src/litert-samples`.
The workspace OpenVINO rule downloads/selects its Android SDK only on a Linux
host; a native Windows Bazel invocation selects the Windows SDK instead.

Docker running a Linux LiteRT build image is another valid option. WSL2 is the
simpler choice when the Android SDK and NDK are already installed there.

## Prerequisites in WSL2

Install the normal LiteRT build tools:

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential clang-18 git openjdk-17-jdk python3 unzip wget zip
```

Install Bazelisk as `bazel`, and install an Android SDK/NDK. Then set:

```bash
export ANDROID_HOME="$HOME/Android/Sdk"
export ANDROID_NDK_HOME="$ANDROID_HOME/ndk/<installed-version>"
```

The workspace registers an Android NDK toolchain and uses API level 26. Its
`.bazelrc` already defines `--config=android_x86_64`.

## First build: repository-pinned OpenVINO

Start with the OpenVINO package pinned by this repository. This proves the
build pipeline before introducing another OpenVINO build.

From the `litert-samples` repository root in WSL:

```bash
bazel build -c opt \
  --config=android_x86_64 \
  --incompatible_enable_cc_toolchain_resolution \
  --incompatible_enable_android_toolchain_resolution \
  --nocheck_visibility \
  @litert_archive//litert/vendors/intel_openvino/compiler:libLiteRtCompilerPlugin_IntelOpenvino.so \
  @litert_archive//litert/vendors/intel_openvino/dispatch:libLiteRtDispatch_IntelOpenvino.so
```

The two `--incompatible_enable_*_toolchain_resolution` flags re-enable modern
Android NDK toolchain selection after this repository's `clang_local` config
disables it. Without them, Bazel can incorrectly ask the Linux host toolchain
for an Android `x86_64` entry and fail in `local_config_cc`.

The root workspace also patches the downloaded LiteRT `main` archive so the
Intel dispatch source includes `npu_hal_wrapper.h`. This fixes the upstream
Android compile error where `NpuHalHooks`, `GetNpuHalHooks`, and priority
helpers are used without their declarations being included.

The exact Bazel output directory can vary for external repositories. Locate the
outputs instead of hard-coding it:

```bash
find bazel-bin -type f \
  \( -name 'libLiteRtCompilerPlugin_IntelOpenvino.so' \
     -o -name 'libLiteRtDispatch_IntelOpenvino.so' \)
```

## Build against your custom OpenVINO

The workspace repository rule accepts `OPENVINO_NATIVE_DIR`. Prepare an overlay
with the layout expected by `third_party/intel_openvino/openvino.bazel`:

```text
/path/to/custom-openvino-repo/
└── openvino_android/
    └── runtime/
        ├── include/
        └── lib/
            └── intel64/
                ├── libopenvino.so
                ├── libopenvino_tensorflow_lite_frontend.so
                ├── libopenvino_intel_npu_plugin.so
                └── ...
```

The names present in a particular OpenVINO build may differ. Preserve its
original Android SDK directory structure and do not substitute host Linux
libraries.

Select the custom SDK and refresh Bazel's external-repository state:

```bash
export OPENVINO_NATIVE_DIR=/absolute/path/to/custom-openvino-repo
bazel clean --expunge
```

Then run the same two-target build command shown above. Both plugins are now
compiled and linked against the OpenVINO headers/libraries selected through
`OPENVINO_NATIVE_DIR`.

If the custom SDK has a different directory layout, update
`third_party/intel_openvino/openvino.bazel` to its actual Android include and
library locations. Do not rename binaries merely to resemble the expected
layout; their SONAME and `DT_NEEDED` entries still control runtime loading.

## Copy the runtime into the app

Copy both generated LiteRT plugins into:

```text
samples/litert/image_segmentation/kotlin_npu/android_jit/
  app/src/main/jniLibs/x86_64/
```

Copy the matching OpenVINO Android `.so` files from:

```text
$OPENVINO_NATIVE_DIR/openvino_android/runtime/lib/intel64/
```

At minimum, expect the OpenVINO runtime, TFLite frontend, and Intel NPU
plugin/compiler. The definitive list is the transitive dependency graph of the
actual files, not a fixed list in this document.

Use the NDK ELF reader to inspect each library:

```bash
READELF="$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-readelf"
"$READELF" -h app/src/main/jniLibs/x86_64/libLiteRtCompilerPlugin_IntelOpenvino.so
"$READELF" -d app/src/main/jniLibs/x86_64/libLiteRtCompilerPlugin_IntelOpenvino.so
"$READELF" -d app/src/main/jniLibs/x86_64/libLiteRtDispatch_IntelOpenvino.so
```

Check that the ELF machine is x86-64 and follow every `NEEDED` entry. Android
system libraries such as `libc.so`, `libdl.so`, `liblog.so`, `libm.so`, and
`libandroid.so` should not be copied. Custom OpenVINO, TBB, and related
non-system dependencies must be packaged.

## Optional: build the matching LiteRT AAR

The app currently uses the released Maven LiteRT `2.1.6`. That is the least
invasive option, but a plugin built from a different LiteRT revision can have
an incompatible plugin ABI.

For full version alignment, build the LiteRT Kotlin AARs from the same LiteRT
source revision used for the Intel plugins:

```bash
bazel build -c opt \
  --config=android_x86_64 \
  --incompatible_enable_cc_toolchain_resolution \
  --incompatible_enable_android_toolchain_resolution \
  --android_ndk_min_sdk_version=24 \
  --define=public_maven_build=true \
  --define=litert_runtime_link_mode=dynamic \
  @litert_archive//litert/kotlin:litert-api-aar \
  @litert_archive//litert/kotlin:litert-aar
```

Do not combine a random LiteRT `main` plugin, Maven `2.1.6`, and an unrelated
OpenVINO nightly for production. Use either:

- LiteRT source/tag compatible with `2.1.6`, plugins built from that source, and
  one matching OpenVINO package; or
- AARs, plugins, and OpenVINO libraries all built from one validated source and
  SDK combination.

Switching this app from Maven to locally built AARs is a separate dependency
change because local AAR files do not carry Maven POM transitive dependencies.
Validate the generated AAR pair before replacing `implementation(libs.litert)`.

## Build and verify the Android app

After all `.so` files have been copied, build the app from the Android project:

```bash
cd samples/litert/image_segmentation/kotlin_npu/android_jit
./gradlew :app:assembleDebug
```

Confirm that the APK contains the custom stack:

```bash
unzip -l app/build/outputs/apk/debug/app-debug.apk | grep 'lib/x86_64/.*\.so'
```

Install it, select NPU, and look for:

```text
LiteRT bundled Intel NPU libraries: /data/app/.../lib/x86_64
```

That path proves the app selected packaged plugins instead of `/vendor/lib64`.
For stronger proof, inspect the process memory map and confirm the OpenVINO
paths begin with `/data/app/`, not `/vendor/lib64`.

## Compatibility boundary

Bundling OpenVINO removes the dependency on the OS OpenVINO user-space runtime.
It does not remove the dependency on the ALOS Intel NPU kernel driver, firmware,
or any device library that the OpenVINO NPU plugin uses to communicate with the
driver. A custom OpenVINO build must remain compatible with that installed
driver/firmware interface.
