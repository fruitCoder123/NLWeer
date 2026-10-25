# Investigation: Docker container for cross-compiling NLWeer

Date: 2026-10-24  
Repository: `CNugteren/NLWeer` / fork `fruitCoder123/NLWeer`  
Scope: investigation only; no source changes were made.

## 1. Executive summary

**Yes, this project can be compiled in a Docker container.** NLWeer is a pure JVM/Android project: Kotlin/Java, XML resources and the Android Gradle Plugin. It has **no native/NDK code**, so no Android NDK, emulator, KVM or device is required to build an APK.

The container needs more than a plain JDK image. It needs:

- Linux `amd64`;
- JDK **17** (the project compiles with Java source/target 17 and `jvmTarget = 17`);
- Android SDK **platform 35** (`compileSdk 35`);
- Android **build-tools 35.x**;
- Android `platform-tools` (optional for building, required for adb);
- `git`, `curl`, `unzip` and standard build tooling;
- network access on first build for Gradle/AGP/Kotlin/Maven dependencies.

The project uses the Gradle wrapper (**Gradle 8.10.2**) and AGP **8.5.0**, Kotlin **1.9.22**.

A ready-made public image exists and is downloadable: **`mingc/android-build-box:latest`** (Docker Hub), which contains JDK 8/11/17/21, Android platforms 28–35 and build-tools 35.0.0. This is the recommended quickest option. A small custom image can also be built from `eclipse-temurin:17-jdk` for a leaner, fully controlled setup.

## 2. Project toolchain requirements

From `build.gradle`, `app/build.gradle`, `gradle.properties` and `gradle/wrapper/gradle-wrapper.properties`:

| Requirement | Value |
|---|---|
| Gradle wrapper | `8.10.2` |
| Android Gradle Plugin | `8.5.0` |
| Kotlin | `1.9.22` |
| `compileSdk` | `35` |
| `targetSdk` | `35` |
| `minSdk` | `21` |
| Java source/target | `17` |
| Kotlin `jvmTarget` | `17` |
| Build features | ViewBinding and BuildConfig |
| Repositories | `google()`, `mavenCentral()`, `jitpack.io` |
| Native/NDK | none |
| Emulator | not needed for `assembleDebug` / `assembleRelease` |

Dependencies are resolved from the network on the first build, including JitPack (`com.github.PhilJay:MPAndroidChart`). The container therefore needs internet access (or a pre-populated Gradle/Maven cache).

## 3. Signing caveat

`app/build.gradle` defines a release signing config that reads:

```gradle
RELEASE_STORE_FILE
RELEASE_STORE_PASSWORD
RELEASE_KEY_ALIAS
RELEASE_KEY_PASSWORD
```

The project `gradle.properties` points these at `~/.android/debug.keystore`. This is enough for **debug** builds, which use the automatically generated debug keystore.

For `assembleRelease` or `./gradlew build` (which includes release assembly), a valid keystore must exist and the paths must be correct. The existing GitHub Actions workflow handles this by creating a temporary keystore and writing absolute paths to `~/.gradle/gradle.properties`. A container build should do the same, or use `assembleDebug` first. Avoid relying on `~` expansion in the value passed to Gradle; use an absolute path in the container.

Example keystore creation for a CI-style build:

```bash
keytool -genkeypair \
  -dname "cn=Docker CI, ou=JavaSoft, o=Sun, c=US" \
  -alias alias -keypass android -keystore /project/ci.keystore \
  -storepass android -keyalg RSA -keysize 2048 -validity 2
```

Then supply `-PRELEASE_STORE_FILE=/project/ci.keystore` (and the other `-P` values), or place them in a mounted `~/.gradle/gradle.properties`.

## 4. Is a suitable image available online?

I verified the image manifests on 2026-10-24 from the Docker CLI. The following images are publicly available:

### 4.1 `mingc/android-build-box:latest` — recommended ready-made image

- Registry: Docker Hub.
- Base: Ubuntu 22.04.
- Java: OpenJDK 8 / 11 / 17 / 21 (managed with jEnv).
- Android platforms: 28–35.
- Build-tools: includes `35.0.0`.
- Also includes NDK, platform-tools, bundletool, fastlane, Flutter, etc.
- Compressed size reported by Docker Hub: about **5.5 GB** for `latest`.
- Manifest fetch succeeded.

This image covers all project requirements and is the simplest way to build. Because it contains multiple JDKs, verify/select JDK 17 before building (for example by selecting it with jEnv or setting `JAVA_HOME` to the image’s JDK 17 path). Gradle 8.10.2 and AGP 8.5.0 are compatible with JDK 17 and also run on newer LTS JDKs, but JDK 17 is the project’s declared target and the safest choice.

Pull and build:

```bash
docker pull mingc/android-build-box:latest

docker run --rm -it \
  -v "$PWD":/project \
  -v "$HOME/.gradle":/root/.gradle \
  -w /project \
  mingc/android-build-box:latest \
  bash -c 'java -version; ./gradlew --no-daemon assembleDebug'
```

Notes:

- Mounting `$HOME/.gradle` preserves the Gradle dependency cache between runs.
- On Linux, the container runs as root by default and can create root-owned files in the project. If that is undesirable, run with `--user "$(id -u):$(id -g)"` and a writable `HOME`/Gradle cache, or fix ownership afterwards.
- The project wrapper downloads Gradle 8.10.2 on first use; the image’s own Gradle installation is not used.
- For release builds, add the keystore configuration described in section 3.

### 4.2 `thyrlian/android-sdk:latest` — minimal Android SDK base

- Registry: Docker Hub.
- Public manifest fetch succeeded.
- Provides Android SDK command-line tools and a base environment, but does **not** necessarily contain platform 35/build-tools 35 or the exact JDK by default.
- Suitable if you want to add the missing SDK packages yourself with `sdkmanager`.
- Requires more setup than `mingc/android-build-box`, but is a good base for a small custom image.

### 4.3 `garretech/android-sdk-image:latest` — SDK 35 image

- Registry: Docker Hub.
- Public, multi-architecture manifest available (`linux/amd64` and `linux/arm64`).
- Contains `platforms;android-35`, `build-tools;35.0.0`, `platform-tools` and older SDKs.
- Good alternative; check that the included JDK is 17 (or install/select it).

### 4.4 `venk0/android-sdk:latest` — CI-oriented SDK image

- Registry: Docker Hub.
- Public manifest available.
- Includes JDK 21, build-tools 35.0.0/34.0.0/…, platform 36 and platform-tools.
- Platform 35 is not guaranteed to be preinstalled; it can be added with `sdkmanager`.
- JDK 21 is workable with Gradle 8.10.2/AGP 8.5, but JDK 17 is the project baseline.

### 4.5 `1ndevelopment/android-build-env:latest` — new 2026 image

- Registry: Docker Hub.
- Public manifest available.
- Contains JDK 25, Gradle 9.3, SDK 35 and build-tools 35.0.0.
- **Not recommended for this project**: JDK 25 is newer than what AGP 8.5 / Gradle 8.10.2 are validated against, and the image is Gradle 9-oriented. It could work if a JDK 17/21 is selected, but it adds unnecessary compatibility risk.

### 4.6 `androidsdk/android-35` — not usable

The name looks like an official Android SDK image, but `docker manifest inspect androidsdk/android-35:latest` was denied / not publicly available during this investigation. Do not depend on it.

## 5. Recommended options

### Option A (fastest): use `mingc/android-build-box`

Pros: everything is preinstalled, reproducible enough for this project, no image build required.  
Cons: large (~5.5 GB compressed), contains many tools that NLWeer does not need, only `linux/amd64`.

Recommended command:

```bash
docker run --rm -it \
  -v "$PWD":/project \
  -v "$HOME/.gradle":/root/.gradle \
  -w /project \
  mingc/android-build-box:latest \
  bash -c './gradlew --no-daemon assembleDebug'
```

### Option B (lean and fully controlled): build a small custom image

Use an official JDK 17 image and install only the Android SDK packages the project needs. This produces a smaller image and avoids tool/version surprises.

Example `Dockerfile`:

```dockerfile
FROM eclipse-temurin:17-jdk-jammy

ENV ANDROID_SDK_ROOT=/opt/android-sdk \
    ANDROID_HOME=/opt/android-sdk \
    DEBIAN_FRONTEND=noninteractive

ENV PATH=$PATH:$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$ANDROID_SDK_ROOT/platform-tools

RUN apt-get update \
    && apt-get install -y --no-install-recommends git curl unzip ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p "$ANDROID_SDK_ROOT/cmdline-tools" \
    && curl -fsSL https://dl.google.com/android/repository/commandlinetools-linux-13114758_latest.zip -o /tmp/cmdline-tools.zip \
    && unzip -q /tmp/cmdline-tools.zip -d "$ANDROID_SDK_ROOT/cmdline-tools" \
    && mv "$ANDROID_SDK_ROOT/cmdline-tools/cmdline-tools" "$ANDROID_SDK_ROOT/cmdline-tools/latest" \
    && rm /tmp/cmdline-tools.zip

RUN yes | sdkmanager --licenses \
    && sdkmanager "platform-tools" "platforms;android-35" "build-tools;35.0.0"

WORKDIR /project
```

Build and use:

```bash
docker build -t nlweer-builder .
docker run --rm -it -v "$PWD":/project -v "$HOME/.gradle":/root/.gradle -w /project nlweer-builder \
  ./gradlew --no-daemon assembleDebug
```

The command-line tools archive version should be pinned to a known good release; `13114758_latest` is a commonly used recent version. Check the Android command-line tools download page for the current filename and update the `Dockerfile` deliberately.

## 6. What is not required

- **No NDK**: there is no `externalNativeBuild`, CMake or native source in the project.
- **No emulator**: `assembleDebug`/`assembleRelease` are pure builds. Instrumented tests (`androidTest`) would need an emulator or a physical device and KVM; that is a separate concern.
- **No Android Studio**: the Gradle wrapper plus Android SDK is sufficient.
- **No Google Play Services**: the app is designed to run without them.

## 7. Practical considerations

- **First build is network-heavy**: Gradle, AGP, Kotlin and all AndroidX/JitPack dependencies are downloaded. Cache `~/.gradle` (and optionally `~/.m2`) between runs.
- **Architecture**: most ready-made Android SDK images are `linux/amd64`. On an ARM host (e.g. Apple Silicon), use `--platform linux/amd64` (slow emulation) or build a native ARM image with the same SDK packages.
- **Licences**: ready-made images usually accept Android SDK licences during their build. A custom image must run `yes | sdkmanager --licenses`, as shown above.
- **Signing secrets**: never bake keystores/passwords into the image. Mount them or pass them at run time.
- **`./gradlew build` vs `assembleDebug`**: `build` also runs lint/tests and assembles release variants, which triggers the release signing config. Start with `assembleDebug` to validate the container, then configure signing for release.
- **File ownership**: on Linux bind mounts, root-created build outputs can be inconvenient. Consider `--user` and a project-local cache, or adjust ownership after the build.
- **Host status**: this host has Docker 29.9.0 and OpenJDK 17 installed, but no Android SDK was detected on `PATH`. Docker is therefore the clean way to get a consistent Android toolchain.

## 8. Verdict

- **Can we create a Docker container to cross-compile this project?** Yes.
- **What kind of image do we need?** A Linux amd64 image with JDK 17 + Android SDK platform 35 + build-tools 35.0.0. A plain JDK image is not enough; an Android SDK image without a compatible JDK is not enough either.
- **Is such an image available for download?** Yes. `mingc/android-build-box:latest` on Docker Hub is the most complete ready-made option and its manifest was verified during this investigation. `thyrlian/android-sdk`, `garretech/android-sdk-image` and `venk0/android-sdk` are viable alternatives. The `androidsdk/android-35` name was not publicly accessible.
- **Recommendation:** use `mingc/android-build-box` for an immediate build; create the small `eclipse-temurin:17-jdk` custom image if a lean, pinned and fully reproducible build environment is preferred.
