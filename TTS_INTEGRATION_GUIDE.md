# Sherpa-ONNX TTS: Library Integration & Model Management

This guide explains how to extract the **sherpa-onnx** TTS engine as a library for any Android application and how to use the existing logic for model downloading and metadata.

---

## 1. Extracting the Library (AAR)

You can use the pre-built Android Archive (AAR) to integrate the engine into your app.

### Option A: JitPack (Recommended)
Add the following to your root `build.gradle` or `settings.gradle`:
```gradle
repositories {
    maven { url 'https://jitpack.io' }
}
```
Add the dependency to your app's `build.gradle`:
```gradle
dependencies {
    implementation 'com.github.k2-fsa:sherpa-onnx:v1.13.2'
}
```

### Option B: Build AAR Manually
If you want to customize the build:
1.  Clone the repo: `git clone https://github.com/k2-fsa/sherpa-onnx`
2.  Navigate to `android/SherpaOnnxAar`.
3.  Download the native JNI libraries (see `android/SherpaOnnxAar/README.md` for `wget` links).
4.  Run `./gradlew :sherpa_onnx:assembleRelease`.
5.  Find the AAR in `sherpa_onnx/build/outputs/aar/`.

---

## 2. TTS Model Metadata & Links

The project maintains a vast list of supported models and their configurations in `scripts/apk/generate-tts-apk-script.py`. 

### Download Base URL
Most models are hosted on GitHub Releases:
`https://github.com/k2-fsa/sherpa-onnx/releases/download/tts-models/{model_dir}.tar.bz2`

### Model Metadata (Key Types)
| Model Type | Metadata / Capabilities | Config Requirements |
| :--- | :--- | :--- |
| **Piper (VITS)** | High quality, 30+ languages. | Requires `espeak-ng-data`. |
| **Kokoro** | Fast, high quality (EN/ZH). | Requires `voices.bin`. |
| **Matcha-TTS** | Fast, requires separate vocoder. | Needs `vocos-22khz-univ.onnx`. |
| **ZipVoice** | Voice cloning capabilities. | Needs encoder/decoder/vocoder. |
| **Supertonic** | 31+ languages support. | Multi-component (.onnx files). |

### Metadata Example (EN/ZH Multi-lingual)
*   **Model Dir:** `kokoro-multi-lang-v1_0`
*   **Languages:** English (en), Chinese (zh)
*   **Files needed:** `model.onnx`, `voices.bin`, `espeak-ng-data`, `lexicon-us-en.txt`, `lexicon-zh.txt`.

---

## 3. Model Download & Extraction Logic

To avoid "inventing the wheel," follow the logic used in the project's CI and application examples:

### Download & Setup Flow
1.  **Download:** Fetch the `{model_dir}.tar.bz2` from the release URL.
2.  **Extract:** Decompress the `.tar.bz2` to your app's internal storage (`context.filesDir`).
3.  **Path Resolution:** Locate the specific files within the extracted directory.
4.  **Vocoder Check:** If using `Matcha-TTS`, download the `vocos` model separately to the root directory.

### Configuration Snippet (Kotlin)
Use this logic to map extracted files to the library's config:

```kotlin
val modelDir = File(context.filesDir, "kokoro-multi-lang-v1_0")

val config = OfflineTtsConfig(
    model = OfflineTtsModelConfig(
        kokoro = OfflineTtsKokoroModelConfig(
            model = "${modelDir.path}/model.onnx",
            voices = "${modelDir.path}/voices.bin",
            tokens = "${modelDir.path}/tokens.txt",
            dataDir = "${modelDir.path}/espeak-ng-data",
            lexicon = "${modelDir.path}/lexicon-us-en.txt,${modelDir.path}/lexicon-zh.txt"
        ),
        numThreads = 1,
        debug = true
    )
)

val tts = OfflineTts(config = config)
```

---

## 4. Implementation Details (from MainActivity.kt)

### Playback Logic
The app uses `AudioTrack` with a streaming callback for low-latency playback:
1.  **Initialize AudioTrack:** Use `AudioFormat.ENCODING_PCM_FLOAT`.
2.  **Callback:** Pass a callback to `tts.generateWithConfigAndCallback`.
3.  **Streaming:** The callback writes samples directly to the `AudioTrack` buffer:
    ```kotlin
    private fun callback(samples: FloatArray): Int {
        track.write(samples, 0, samples.size, AudioTrack.WRITE_BLOCKING)
        return 1 // Continue generation
    }
    ```

### Asset vs File Storage
*   If models are in `assets/`, use `OfflineTts(assetManager, config)`.
*   If models are downloaded to storage, use `OfflineTts(config = config)`.

---

## 5. Building a Static-ORT AAR (Resolving ORT Version Conflicts)

If your app already ships `libonnxruntime.so` from a **different ORT version** than the one Sherpa was compiled against, the Android linker will refuse to load `libsherpa-onnx-jni.so` at runtime with an error like:

```
UnsatisfiedLinkError: cannot locate symbol "OrtGetApiBase" referenced by libsherpa-onnx-jni.so
```

This happens because each ORT version exports a version-tagged symbol (`VERS_1.23.0`, `VERS_1.24.3`, etc.) and the linker enforces an exact match. The pre-built Sherpa 1.13.2 AAR requires `VERS_1.24.3`.

**Root cause:** GNU symbol versioning — the linker checks not just that `OrtGetApiBase` exists but that it is tagged with the exact version string that was present at compile time.

**Solution: build `libsherpa-onnx-jni.so` with `BUILD_SHARED_LIBS=OFF`.**  
This statically links ORT into the JNI library. The resulting `.so` has no `DT_NEEDED` on `libonnxruntime.so` at all, so no version conflict is possible regardless of what ORT version the rest of the app uses.

### Step 1 — Build from source with static ORT

```bash
cd /path/to/sherpa-onnx
ANDROID_NDK=/path/to/ndk/27.x.x BUILD_SHARED_LIBS=OFF ./build-android-arm64-v8a.sh
```

- Downloads `onnxruntime-android-arm64-v8a-static_lib-{version}.zip` automatically.
- Output: `build-android-arm64-v8a-static/install/lib/libsherpa-onnx-jni.so`
- The c-api and cxx-api shared libs are **not** built (they are not needed for JNI/Android use).

### Step 2 — Swap libs in the AAR project

```bash
JNIDIR=android/SherpaOnnxAar/sherpa_onnx/src/main/jniLibs/arm64-v8a

# Remove libs that have DT_NEEDED on libonnxruntime.so
rm $JNIDIR/libonnxruntime.so
rm $JNIDIR/libsherpa-onnx-c-api.so
rm $JNIDIR/libsherpa-onnx-cxx-api.so

# Replace the JNI lib with the static build
cp build-android-arm64-v8a-static/install/lib/libsherpa-onnx-jni.so $JNIDIR/
```

### Step 3 — Assemble the AAR

```bash
cd android/SherpaOnnxAar
./gradlew :sherpa_onnx:assembleRelease
cp sherpa_onnx/build/outputs/aar/sherpa_onnx-release.aar ../../sherpa-onnx-{version}-static.aar
```

### Step 4 — Wire up in your app

Reference the local AAR in your module's `build.gradle.kts`:

```kotlin
implementation(files("/path/to/sherpa-onnx-{version}-static.aar"))
```

The `pickFirsts += "lib/*/libonnxruntime.so"` rule in your app's packaging config is no longer needed for the Sherpa AAR (since it no longer ships `libonnxruntime.so`), but it is harmless to leave in place.

### Notes for future Sherpa upgrades

- Re-run these steps for each new Sherpa version.
- The ORT version bundled inside the static `.so` is whatever `onnxruntime_version` is set to in `build-android-arm64-v8a.sh` — it has no impact on the ORT version used by the rest of the app.
- If you want Sherpa to use the **same** ORT version as your app (shared libs, smaller APK), change `onnxruntime_version` in the build script to match, then build with `BUILD_SHARED_LIBS=ON`. The resulting `.so` will require the matching `VERS_X.Y.Z` tag.

---

## 6. Summary of Resources
*   **Models Release Page:** [k2-fsa/sherpa-onnx/releases/tag/tts-models](https://github.com/k2-fsa/sherpa-onnx/releases/tag/tts-models)
*   **AAR Project:** `android/SherpaOnnxAar`
*   **TTS Demo App:** `android/SherpaOnnxTts`
*   **Binding Code:** `android/SherpaOnnxAar/sherpa_onnx/src/main/java/com/k2fsa/sherpa/onnx/Tts.kt`
