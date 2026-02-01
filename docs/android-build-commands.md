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

The workflow file at `.github/workflows/android-release.yml` automates the build and release process.

### Prerequisites

Before using the GitHub Actions workflow, set up the required secrets (see "Signing the APK" section above):
- `ANDROID_KEYSTORE_BASE64`
- `ANDROID_KEYSTORE_PASSWORD`
- `ANDROID_KEY_PASSWORD`
- `ANDROID_KEY_ALIAS`

### Triggering a Build

1. **Push a version tag** (recommended for releases):
   ```bash
   git tag v1.0.0
   git push origin v1.0.0
   ```

2. **Manual trigger**: Go to Actions tab in GitHub and run the workflow manually

### What the Workflow Does

The automated workflow will:
1. Set up all prerequisites (Java, Node.js, Rust, Android SDK/NDK)
2. Install dependencies and build the frontend
3. Build the Android APK (ARM64 architecture)
4. **Sign the APK** (when triggered by a tag)
5. Upload signed artifacts
6. Create a GitHub release with the signed APK attached

### Workflow Outputs

- **Unsigned APK** (for development/testing): Available as workflow artifact
- **Signed APK** (for release): Attached to GitHub release

### Permissions

The workflow requires `contents: write` permission to create releases. This is automatically configured in the workflow file.

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

### Step 1: Generate Release Keystore (One-time Setup)

```bash
# Generate keystore
keytool -genkey -v -keystore release-keystore.jks \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -alias release-key \
  -dname "CN=ericwang, OU=onlyquant, O=onlyquant, L=shanghai, ST=shanghai, C=CN"

# You'll be prompted for:
# - Keystore password (remember this!)
# - Key password (can be same as keystore password)
```

**Important**: Keep your keystore file (`release-keystore.jks`) and passwords **secure**! If you lose them, you cannot update your app on the Play Store.

### Step 2: Sign APK Locally

```bash
# Navigate to APK directory
cd client/src-tauri/gen/android/app/build/outputs/apk/universal/release/

# Sign the APK
jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 \
  -keystore /path/to/release-keystore.jks \
  app-universal-release-unsigned.apk \
  release-key

# Align the APK (optimizes for Android)
$ANDROID_HOME/build-tools/34.0.0/zipalign -v 4 \
  app-universal-release-unsigned.apk \
  app-universal-release.apk

# Verify the signature
jarsigner -verify -verbose -certs app-universal-release.apk
```

### Step 3: Setup GitHub Secrets for Automated Signing

To enable automatic signing in GitHub Actions, add these secrets to your repository:

1. Go to your GitHub repository → Settings → Secrets and variables → Actions
2. Add the following secrets:

**Required Secrets:**

- `ANDROID_KEYSTORE_BASE64`: Your keystore file encoded in base64
  ```bash
  base64 -w 0 release-keystore.jks > keystore.base64.txt
  # Copy the contents and add as secret
  ```

- `ANDROID_KEYSTORE_PASSWORD`: The password you used for the keystore

- `ANDROID_KEY_PASSWORD`: The password for the key alias (often same as keystore password)

- `ANDROID_KEY_ALIAS`: The alias name (e.g., `release-key`)

**How to encode keystore:**
```bash
# On Linux/Mac
base64 -w 0 release-keystore.jks

# On Windows (PowerShell)
[Convert]::ToBase64String([IO.File]::ReadAllBytes("release-keystore.jks"))
```

### Step 4: Verify Signed APK

After signing, verify the APK:

```bash
# Check signature
jarsigner -verify -verbose -certs app-universal-release.apk

# View APK details
aapt dump badging app-universal-release.apk | grep package

# Check certificate
keytool -printcert -jarfile app-universal-release.apk
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
