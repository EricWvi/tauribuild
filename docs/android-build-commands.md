# Tauri Android Build - Command Reference

This document contains all the Linux commands needed to build Tauri Android APKs, suitable for use in GitHub Actions or local development.

## Prerequisites Installation

### 1. Install Rust
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
```

### 2. Install Android SDK and NDK
```bash
# Create Android SDK directory
mkdir -p $HOME/Android/cmdline-tools
cd $HOME/Android/cmdline-tools

# Download Android command line tools
wget https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip -O cmdline-tools.zip

# Extract and organize
unzip -q cmdline-tools.zip
mkdir -p latest
mv cmdline-tools/* latest/
rm cmdline-tools.zip

# Set up environment
export ANDROID_HOME=$HOME/Android
export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools

# Accept licenses
yes | sdkmanager --licenses

# Install required SDK components
sdkmanager "platform-tools" "platforms;android-34" "build-tools;34.0.0" "ndk;27.2.12479018"

# Set NDK_HOME
export NDK_HOME=$ANDROID_HOME/ndk/27.2.12479018
```

### 3. Persist Environment Variables
```bash
echo 'export ANDROID_HOME=$HOME/Android' >> ~/.bashrc
echo 'export NDK_HOME=$ANDROID_HOME/ndk/27.2.12479018' >> ~/.bashrc
echo 'export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools' >> ~/.bashrc
source ~/.bashrc
```

### 4. Install Java 17 (using SDKMAN)
```bash
# If using SDKMAN
sdk install java 17.0.17-amzn
sdk use java 17.0.17-amzn

# Or using apt (Ubuntu/Debian)
# sudo apt update
# sudo apt install -y openjdk-17-jdk
```

### 5. Install Rust Android Targets
```bash
rustup target add aarch64-linux-android
rustup target add armv7-linux-androideabi
rustup target add i686-linux-android
rustup target add x86_64-linux-android
```

## Project Setup

### 1. Install Tauri Dependencies
```bash
cd /path/to/your/project/client
npm install -D @tauri-apps/cli@next @tauri-apps/api@next
```

### 2. Add Tauri Script to package.json
Add to your `package.json` scripts:
```json
{
  "scripts": {
    "tauri": "tauri"
  }
}
```

### 3. Initialize Tauri (if not already done)
Create `src-tauri` directory with necessary files, or let Tauri generate it.

### 4. Initialize Android Support
```bash
cd /path/to/your/project/client
npm run tauri android init
```

## Building the APK

### Build for Single Architecture (ARM64)
```bash
cd /path/to/your/project/client
npm run build  # Build frontend first
npm run tauri android build -- --target aarch64
```

### Build for All Architectures
```bash
npm run tauri android build
```

### Output Locations
- **APK**: `client/src-tauri/gen/android/app/build/outputs/apk/universal/release/app-universal-release-unsigned.apk`
- **AAB**: `client/src-tauri/gen/android/app/build/outputs/bundle/universalRelease/app-universal-release.aab`

## Troubleshooting

### Java Version Issues
Gradle may not support the latest Java versions. Use Java 17:
```bash
sdk use java 17.0.17-amzn
# or
export JAVA_HOME=/path/to/java-17
```

### Memory Issues
If Gradle daemon crashes, build for a single architecture:
```bash
npm run tauri android build -- --target aarch64
```

### Rebuild Android Project
If you change the app identifier, regenerate the Android project:
```bash
rm -rf client/src-tauri/gen/android
npm run tauri android init
```

## GitHub Actions Usage

The workflow file at `.github/workflows/android-release.yml` automates this process. To trigger:

1. **Push a version tag**:
   ```bash
   git tag v1.0.0
   git push origin v1.0.0
   ```

2. **Manual trigger**: Go to Actions tab in GitHub and run the workflow manually

The workflow will:
- Set up all prerequisites
- Build the APK
- Upload it as an artifact
- Create a GitHub release (for tags)

## Configuration Files

### tauri.conf.json
Located at `client/src-tauri/tauri.conf.json`:
- Set unique `identifier` (e.g., `com.yourcompany.app`)
- Configure `frontendDist` to match your build output
- Set `devUrl` to match your dev server

### Gradle Settings
If needed, update Java compatibility in `client/src-tauri/gen/android/app/build.gradle.kts`:
```kotlin
compileOptions {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

kotlinOptions {
    jvmTarget = "17"
}
```

## Signing the APK (Production)

For distribution, sign your APK:

```bash
# Generate keystore (first time only)
keytool -genkey -v -keystore my-release-key.jks -keyalg RSA -keysize 2048 -validity 10000 -alias my-key-alias

# Sign the APK
jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 -keystore my-release-key.jks app-universal-release-unsigned.apk my-key-alias

# Align the APK
zipalign -v 4 app-universal-release-unsigned.apk app-release.apk
```

## Verification

Check the APK was created:
```bash
ls -lh client/src-tauri/gen/android/app/build/outputs/apk/universal/release/
```

Inspect APK contents:
```bash
aapt dump badging app-universal-release-unsigned.apk
```
