# Bundled Intel OpenVINO libraries

Place Android x86_64 shared libraries in this directory. Gradle packages `.so`
files from here into the APK; this README is not packaged as a native library.

Required LiteRT entry points:

- `libLiteRtCompilerPlugin_IntelOpenvino.so`
- `libLiteRtDispatch_IntelOpenvino.so`

Also copy the matching Android OpenVINO runtime, TensorFlow Lite frontend, Intel
NPU plugin/compiler, and every non-system `DT_NEEDED` dependency. Do not copy
Linux host libraries: all files here must target Android x86_64.

See `../../../../../CUSTOM_OPENVINO_BUILD.md` for build and verification steps.
