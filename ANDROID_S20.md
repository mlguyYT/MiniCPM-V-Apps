# Experimental MiniCPM-V 4.7 Android Setup

This ML Guy fork contains changes used to run MiniCPM-V 4.7 on a Samsung
Galaxy S20. It is an experimental adaptation of OpenBMB's app, not an official
OpenBMB release or a promise of support for every Android phone.

Upstream app baseline: `fa9d837e3d38dfde7a9484ec1cc6bc2e5bc9b420`.
The engine submodule points at the matching patched commit in
[mlguyYT/llama.cpp-omni](https://github.com/mlguyYT/llama.cpp-omni/tree/ios-omni).
The model weights and licenses remain with their publishers.

## What Has Actually Worked

Tested 2026-10-15 on a Galaxy S20 SM-G980F, Android 13, ARM64, 8 GB advertised
RAM, using CPU inference without Samsung NPU or GPU acceleration:

| Check | Result |
| --- | --- |
| MiniCPM-V 4.7 1B | Text replies worked on an earlier local build |
| Image selection | Test screenshot attached through Android's picker |
| MiniCPM-V 4.7 35B-A3B | Main model, expert runtime and vision projector loaded |
| 35B text request | `Reply with exactly READY.` produced `READY` |
| Short-request elapsed time | About 12 seconds including prompt processing |

This tiny request is not a meaningful sustained throughput benchmark. Image
question answering, offline-only operation, longer conversations, battery,
heat, other devices and final-build 1B regression remain unverified. Significant
Android swap use was observed. No iPhone memory or throughput claim has been
reproduced, and DSpark is not integrated in this build.

## Changes In This Fork

### Model Download And Setup

- Added 4.7 1B and experimental 35B model choices and their published mobile
  artifact paths. The first/default entry for a fresh setup is the 1B model.
- Added manifest-driven downloads and expected-length checks for the 35B
  model's 40 expert files. A main GGUF alone is not the complete download.
- Connected the Android JNI bridge to the engine's existing expert streaming
  configuration and set the model's internal template/version identifiers.

The app reuses its existing Hugging Face and ModelScope download sources.
The 1B artifacts total roughly 1.8 GB. The 35B set totals roughly 16.6 GB,
including the main model, vision projector and 40 expert files.
Expected file lengths detect missing or truncated expert files; they do not
prove content integrity. Full published-checksum verification of the 35B set
was not performed.

### CPU Expert Loading

The initial 35B load reported:

```text
backend has no alloc-only buft, cannot synthesize stripped expert tensor
```

The engine needed writable working slots for experts stored outside the main
GGUF. The inspected upstream path supplied that allocation for Metal, but
lacked the equivalent CPU hook/registration. The engine fork adds it and uses
the unique buffer name `CPU_Expert_Slots` to preserve a separate allocation.
It also guards Apple's optional `F_NOCACHE` API on Android.

See [the engine notes](https://github.com/mlguyYT/llama.cpp-omni/blob/ios-omni/ANDROID_CPU_EXPERTS.md)
for the exact scope. Expert scheduling, model architecture and quantization
were provided by upstream; this fork does not invent or replace them.

### Image Formatting And Failed-Load Cleanup

- Pair an imported image with the following user message in the 4.7 template.
- Use the 4.7 assistant prefix with a closed thinking section.
- Free partially loaded native resources on failure and clear freed globals.
- Avoid clearing an absent inference context and rearming repeated automatic
  load attempts after failure. Manual retries remain in the existing UI.

Intermediate broken retries were followed by Android low-memory kills. The
original missing-buffer error was a separate software failure. Controlled
fault-injection testing of the revised cleanup has not been performed.

## Clone And Build

```sh
git clone --recurse-submodules --shallow-submodules https://github.com/mlguyYT/MiniCPM-V-Apps.git
cd MiniCPM-V-Apps
git submodule status
```

Use the pinned engine commit. Do not replace it with the engine fork's
`master` branch; the Android fixes are on its `ios-omni` branch.

For an existing clone after updating the parent repository:

```sh
git submodule sync
git submodule update --init --recursive
```

Use Android Studio with the project's compatible JDK and SDK. The checked-in
Gradle settings select Android API 36.1, NDK `27.0.12077973`, CMake `4.1.2`
and the `arm64-v8a` ABI. The older upstream README's CMake version description
does not override these Gradle settings.

```sh
cd MiniCPM-V-demo-Android
./gradlew assembleDebug
```

A fresh checkout build on a standard development host has not been verified.
The device-tested APK was built on an ARM64 RUBIK Pi with native ARM64
CMake/Ninja/Clang/LLD and NDK Android headers/runtimes after the x86-64 host
tool path failed. That local host-tool replacement is not automated by this
fork. Building on a host directly supported by the NDK avoids that particular
host-architecture detour.

### Native Library Packaging Check

The tested setup used a packaging-only Gradle invocation after native
compilation. That path initially omitted dependencies, causing
`UnsatisfiedLinkError: library libmtmd.so not found`. For that path, native
dependencies were copied from the matching native build into
`app/src/main/jniLibs/arm64-v8a/` before packaging.

The tested APK included:

```text
libminicpm_v_demo.so
libmtmd.so
libllama.so
libllama-common.so
libggml.so
libggml-cpu.so
libggml-base.so
libomp.so
libggml-cpu-v86.so
```

`libomp.so` came from the ARM64 Android NDK runtime, not the build host's
Linux libraries. The v86 variant is produced by the existing Gradle task;
the tested S20 selected the baseline CPU backend. Check APK contents if
reproducing the packaging-only build; skipping native tasks does not build
or collect missing libraries for you. Build outputs and local SDK paths are
intentionally not committed.

## Using The App

Open Model Management, select a model, download the full set and load it.
Text/image controls stay disabled until the engine is ready. For the 35B
model, wait for all expert files to finish; the download can take substantial
time and storage. Upstream recommends 12 GB RAM for this model; the short
8 GB S20 test does not establish a lower universal requirement.

Settings used for the verified 35B reply:

| Setting | Value |
| --- | --- |
| GPU layers | 0 |
| Context capacity | 4096 |
| Logical batch capacity | 2048 |
| Compute microbatch | 8 |
| Shared prefill slots | 64 |
| Decode cache setting | 8 |
| Lookahead depth | 4 |

The expert runtime also allocates per-layer windows; the complete expert
working allocation is larger than 64 expert slots. These are tested settings,
not benchmark-selected optima.

The app retains upstream's `com.tianchi.minicpmv` application ID. A differently
signed build cannot necessarily update an existing installation. Do not
uninstall an existing app casually: doing so can delete its model downloads
and local data. Keep signing keys private. No binary is published by this fork
yet, and the existing upstream version fields have not been bumped.

## Validation And Attribution

Native compilation, Gradle APK assembly, source whitespace checks and APK v2
signature verification passed for the device-tested build. That build loaded
the 35B model and completed the short text request described above. There has
been no full upstream CI run, iOS/HarmonyOS regression, perplexity comparison
or broad device test matrix.

The model and core inference runtime remain the work of their upstream
authors. Existing MIT notices are preserved; downloaded model weights have
separate terms. The local fixes were developed with coding-agent assistance
and require human review before any upstream contribution. No upstream pull
request has been submitted.
