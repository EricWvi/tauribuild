# Development Log

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
