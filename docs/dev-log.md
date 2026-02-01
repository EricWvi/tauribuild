# Development Log

## 2026-02-01 - Android SDK Upgrade to Version 36

### Updated Android SDK from 34 to 36

**Changes Implemented**:
- Updated `compileSdk` from 34 to 36 in [client/src-tauri/gen/android/app/build.gradle.kts](client/src-tauri/gen/android/app/build.gradle.kts#L16)
- Updated `targetSdk` from 34 to 36 in [client/src-tauri/gen/android/app/build.gradle.kts](client/src-tauri/gen/android/app/build.gradle.kts#L29)
- Updated build-tools version from 34.0.0 to 36.0.0 in [.github/workflows/android-release.yml](.github/workflows/android-release.yml) for apksigner

**Files Modified**:
- [client/src-tauri/gen/android/app/build.gradle.kts](client/src-tauri/gen/android/app/build.gradle.kts) - Updated compileSdk and targetSdk
- [.github/workflows/android-release.yml](.github/workflows/android-release.yml) - Updated build-tools version for APK signing

**Testing**: Verified that Vite dev server starts successfully without errors.

---

## 2026-02-01 - Night Update

### Fixed APK Installation Error - Switched to apksigner

**Issue**: APK installation was failing on Android devices with error:
```
INSTALL_FAILED_INVALID_APK: Failed to extract native libraries, res=-2
```

**Root Cause**: The signing process was using the deprecated `jarsigner` tool with incorrect ordering. The workflow was signing first, then aligning, which causes corruption. With `jarsigner`, you must align **before** signing, not after.

**Solution**: Switched to `apksigner` (the modern Android signing tool) which:
- Handles both signing and alignment automatically in the correct order
- Uses modern v2/v3 APK signature schemes
- Is the recommended tool by Google for APK signing

**Files Modified**:
- [.github/workflows/android-release.yml](.github/workflows/android-release.yml) - Replaced jarsigner/zipalign with apksigner
- [docs/android-build-commands.md](android-build-commands.md) - Updated documentation with apksigner as recommended method

**Verification**: APK now signs correctly with v2 and v3 signature schemes verified.

---

## 2026-02-01 - Late Evening Update

### Fixed Keystore Alias Mismatch in GitHub Actions

**Issue**: GitHub Actions build was failing during APK signing with error:
```
jarsigner: Certificate chain not found for: release-keystore.
release-keystore must reference a valid KeyStore key entry containing 
a private key and corresponding public key certificate chain.
```

**Root Cause**: The jarsigner command was using the wrong alias name. The keystore was created with alias `release-key` (as documented in android-build-commands.md), but the workflow was trying to sign with alias `release-keystore`.

**Solution**: Updated [.github/workflows/android-release.yml](.github/workflows/android-release.yml#L85) to use the correct alias `release-key`.

**Files Modified**:
- [.github/workflows/android-release.yml](.github/workflows/android-release.yml) - Changed jarsigner alias from `release-keystore` to `release-key`
- [docs/todo.md](todo.md) - Marked build issue as resolved

**Note**: The deprecated `set-output` warning mentioned in the todo was not applicable - the current workflow doesn't use this deprecated command.

---

## 2026-02-01 - Evening Update

### Fixed GitHub Actions Build Issues & Added APK Signing

Resolved the 403 permission error and implemented automated APK signing in the GitHub Actions workflow.

#### Issues Fixed

**1. GitHub Actions 403 Permission Error**
- **Problem**: `softprops/action-gh-release@v1` failed with 403 status when attempting to create releases
- **Root Cause**: Missing write permissions for repository contents
- **Solution**: Added `permissions: contents: write` to workflow file
- **File Modified**: [.github/workflows/android-release.yml](.github/workflows/android-release.yml)

**2. Unsigned APK Release**
- **Problem**: APKs were being released unsigned, unsuitable for distribution
- **Solution**: Implemented automated signing workflow using GitHub Secrets
- **Implementation**:
  - Added keystore setup step to decode base64-encoded keystore from secrets
  - Sign APK with `jarsigner` using SHA256withRSA algorithm
  - Align APK with `zipalign` for optimization
  - Verify signature after signing

#### Workflow Changes

**New Steps Added:**
1. **Setup Keystore**: Decodes base64 keystore from `ANDROID_KEYSTORE_BASE64` secret
2. **Sign APK**: Signs using keystore credentials from GitHub secrets
3. **Upload Signed Artifacts**: Updated to upload signed APK instead of unsigned

**Required GitHub Secrets:**
- `ANDROID_KEYSTORE_BASE64`: Base64-encoded keystore file
- `ANDROID_KEYSTORE_PASSWORD`: Keystore password
- `ANDROID_KEY_PASSWORD`: Key alias password
- `ANDROID_KEY_ALIAS`: Key alias name (e.g., "release-key")

**Signing Process:**
```bash
# Sign APK
jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 \
  -keystore release-keystore.jks \
  app-universal-release-unsigned.apk release-key

# Align APK
zipalign -v 4 app-universal-release-unsigned.apk app-universal-release.apk

# Verify
jarsigner -verify -verbose -certs app-universal-release.apk
```

#### Documentation Updates

Updated [docs/android-build-commands.md](docs/android-build-commands.md) with:
- Comprehensive keystore generation guide
- Step-by-step local APK signing instructions
- GitHub Secrets setup for automated signing
- Base64 encoding instructions for keystore
- APK verification commands
- Enhanced GitHub Actions usage documentation

#### Security Notes

- Signing only occurs on tagged releases (not manual workflow runs without tags)
- Keystore credentials stored securely in GitHub Secrets
- Unsigned APKs still available as artifacts for testing

#### Next Steps for Users

To enable signing in your fork/repository:
1. Generate a release keystore using `keytool`
2. Encode keystore to base64
3. Add the 4 required secrets to GitHub repository settings
4. Push a version tag to trigger signed release build

---

## 2026-02-01

### Tauri Android Build Setup - Complete Implementation

Successfully set up Tauri Android build system for creating APKs from a React + Vite client application.

#### Prerequisites Installation
- **Rust**: Installed Rust 1.93.0 via rustup for Tauri backend compilation
- **Android SDK**: Downloaded and configured Android command line tools
  - Platform Tools, Android Platform 34/36, Build Tools 34.0.0
  - NDK 27.2.12479018
- **Java**: Installed Java 17 (Corretto 17.0.17) for Gradle compatibility
  - Note: Java 25 initially present but incompatible with Gradle
- **Android Rust Targets**: Added cross-compilation targets (aarch64, armv7, i686, x86_64)

#### Environment Configuration
- Set `ANDROID_HOME` to `$HOME/Android`
- Set `NDK_HOME` to `$ANDROID_HOME/ndk/27.2.12479018`
- Persisted environment variables to `.bashrc`

#### Tauri Project Setup
- Added `@tauri-apps/cli@next` and `@tauri-apps/api@next` dependencies
- Created `src-tauri` directory with Rust backend
- Configured `tauri.conf.json` with app identifier `com.tauribuild.app`
- Initialized Android support with `tauri android init`

#### Build Process
- Successfully built Android APK (9.1MB) for ARM64 architecture
- Output: `client/src-tauri/gen/android/app/build/outputs/apk/universal/release/app-universal-release-unsigned.apk`
- Fixed Gradle compatibility: Updated to 8.12.1, set Java/Kotlin target to 17

#### GitHub Actions Workflow
- Created `.github/workflows/android-release.yml` for automated CI/CD
- Triggers on version tags or manual dispatch
- Builds APK and uploads to GitHub releases

#### Build Commands
```bash
cd client
npm run tauri android build -- --target aarch64
```

#### Technical Notes
- Gradle 8.12.1 requires Java 17 (Java 25 not yet supported)
- Building multiple architectures simultaneously may cause memory issues in constrained environments
- APK is unsigned; requires signing for distribution

---

### Added Subtle Gradient Background
- Added a subtle top-to-bottom color transition to the page background
- Dark mode: Gradient from `#242424` to `#1a1a1a`
- Light mode: Gradient from `#ffffff` to `#f0f0f0`
