# TimeForge Booster - Complete APK Signing & Release Guide

## Prerequisites

- **Android Studio** or **Build Tools**
- **Java Development Kit (JDK) 17+**
- **Unsigned Release APK** from GitHub Actions

---

## Step 1: Create a Keystore (One-time Setup)

### Option A: Using Android Studio
1. Open Android Studio
2. Go to **Build > Generate Signed Bundle/APK**
3. Select **APK**
4. Click **Create new...** to generate a new keystore
5. Fill in the details:
   - **Key store path**: Save location (e.g., `~/timeforge.jks`)
   - **Password**: Create a strong password
   - **Key alias**: Name for this key (e.g., `timeforge-release-key`)
   - **Key password**: Can be same as keystore password
   - **Validity (years)**: 25+ (for Play Store)
   - **Certificate fields**: Fill your organization details

### Option B: Using Command Line
```bash
keytool -genkey -v -keystore ~/timeforge.jks \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -alias timeforge-release-key
```

**Save these details securely:**
- Keystore path: `~/timeforge.jks`
- Keystore password: `YOUR_KEYSTORE_PASSWORD`
- Key alias: `timeforge-release-key`
- Key password: `YOUR_KEY_PASSWORD`

---

## Step 2: Sign the Release APK

### Download the Unsigned APK
1. Go to [GitHub Actions](https://github.com/srikantsinghray-source/paperclip/actions)
2. Click the latest successful build
3. Download `timeforge-booster-release-unsigned` artifact
4. Extract and locate `app-release-unsigned.apk`

### Sign Using jarsigner (Command Line)
```bash
jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 \
  -keystore ~/timeforge.jks \
  app-release-unsigned.apk timeforge-release-key
```

When prompted, enter your keystore password.

### Sign Using Android Studio
1. Go to **Build > Generate Signed Bundle/APK**
2. Select **APK**
3. Choose your keystore file (`timeforge.jks`)
4. Enter credentials
5. Select **release** build variant
6. Click **Finish**

Output: `app-release.apk` (signed)

---

## Step 3: Optimize with Zipalign

```bash
# Install Build Tools if needed
sdkmanager "build-tools;34.0.0"

# Zipalign the APK
zipalign -v 4 app-release.apk app-release-optimized.apk

# Verify alignment
zipalign -c -v 4 app-release-optimized.apk
```

**Result**: `app-release-optimized.apk` (final APK ready for Play Store)

---

## Step 4: Verify APK Signature

### Verify Signature
```bash
jarsigner -verify -verbose -certs app-release-optimized.apk
```

Expected output:
```
sm       1629 Wed Jun 07 14:00:00 UTC 2026 AndroidManifest.xml
         X.509, CN=Your Name, O=Your Org, L=Your City, ST=Your State, C=Your Country
         [certificate is valid]
jar verified.
```

### Get Certificate Details
```bash
keytool -printcert -jarfile app-release-optimized.apk
```

---

## Step 5: Upload to Google Play Console

### Setup Play Console
1. Go to [Google Play Console](https://play.google.com/console)
2. Create or select your app
3. Navigate to **Release > Production**

### Upload APK
1. Click **Create new release**
2. Upload your signed `app-release-optimized.apk`
3. Add release notes
4. Review and publish

### Important Notes
- **First upload**: Play Console will register your app's signing certificate
- **Future updates**: Must use the SAME keystore (keep it safe!)
- **Backup keystore**: Store keystore in a secure location with password backup

---

## Automated Signing with GitHub Actions

### Step 1: Encode Keystore to Base64
```bash
base64 -i ~/timeforge.jks -o keystore.b64
cat keystore.b64
```
Copy the entire output.

### Step 2: Add GitHub Secrets
1. Go to repo **Settings > Secrets and variables > Actions**
2. Click **New repository secret**
3. Add the following secrets:

| Secret Name | Value |
|-----------|-------|
| `KEYSTORE_BASE64` | (Paste the base64 output) |
| `KEYSTORE_PASSWORD` | Your keystore password |
| `KEY_ALIAS` | `timeforge-release-key` |
| `KEY_PASSWORD` | Your key password |

### Step 3: Update Workflow

Create `.github/workflows/build-signed-release.yml`:

```yaml
name: Build Signed Release APK

on:
  workflow_dispatch:
    inputs:
      versionName:
        description: 'Version name (e.g., 1.0.0)'
        required: true
        default: '1.0.0'

jobs:
  build-signed:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: gradle
    
    - name: Install NDK
      run: |
        $ANDROID_SDK_ROOT/cmdline-tools/latest/bin/sdkmanager --sdk_root=$ANDROID_SDK_ROOT "ndk;25.1.8937393"
    
    - name: Decode Keystore
      run: |
        echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > keystore.jks
    
    - name: Grant execute permission
      run: chmod +x gradlew
    
    - name: Build Signed Release APK
      run: ./gradlew assembleRelease --stacktrace
      env:
        KEYSTORE_PATH: ${{ github.workspace }}/keystore.jks
        KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
        KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
        KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
    
    - name: Zipalign APK
      run: |
        $ANDROID_SDK_ROOT/build-tools/34.0.0/zipalign -v 4 \
          app/build/outputs/apk/release/app-release.apk \
          app/build/outputs/apk/release/app-release-optimized.apk
    
    - name: Upload Signed APK
      uses: actions/upload-artifact@v4
      with:
        name: timeforge-booster-signed
        path: app/build/outputs/apk/release/app-release-optimized.apk
        retention-days: 90
    
    - name: Create GitHub Release
      uses: softprops/action-gh-release@v1
      with:
        files: app/build/outputs/apk/release/app-release-optimized.apk
        draft: true
        prerelease: false
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Step 4: Update app/build.gradle.kts

Uncomment and configure signing:

```kotlin
signingConfigs {
    create("release") {
        storeFile = file(System.getenv("KEYSTORE_PATH") ?: "keystore.jks")
        storePassword = System.getenv("KEYSTORE_PASSWORD") ?: ""
        keyAlias = System.getenv("KEY_ALIAS") ?: ""
        keyPassword = System.getenv("KEY_PASSWORD") ?: ""
    }
}

buildTypes {
    release {
        isMinifyEnabled = true
        isShrinkResources = true
        proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        signingConfig = signingConfigs.getByName("release")
    }
}
```

---

## Quick Reference Commands

```bash
# Generate keystore
keytool -genkey -v -keystore ~/timeforge.jks -keyalg RSA -keysize 2048 -validity 10000 -alias timeforge-release-key

# Sign APK
jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 -keystore ~/timeforge.jks app-release-unsigned.apk timeforge-release-key

# Zipalign
zipalign -v 4 app-release.apk app-release-optimized.apk

# Verify signature
jarsigner -verify -verbose -certs app-release-optimized.apk

# Get certificate fingerprint (for Play Console)
keytool -list -v -keystore ~/timeforge.jks -alias timeforge-release-key
```

---

## Troubleshooting

### Error: "Keystore was tampered with or password was incorrect"
- Re-enter the correct keystore password
- Verify keystore file exists and is readable

### Error: "jarsigner: unable to open jar file"
- Ensure APK file path is correct
- APK must be in the same directory or provide full path

### Error: "Certificate chain is not valid"
- Regenerate keystore with extended validity (25+ years)
- Ensure certificate Common Name (CN) is set correctly

### APK Won't Install
- Verify signing certificate matches previous versions
- Check device date/time (certificate must not be expired)
- Uninstall previous version before installing signed APK

---

## Security Best Practices

✅ **DO:**
- Keep keystore file in a safe location
- Use strong passwords (20+ characters)
- Backup keystore and passwords securely
- Store keystore outside of version control (add to `.gitignore`)
- Use environment variables for credentials in CI/CD

❌ **DON'T:**
- Commit keystore to Git repository
- Share keystore password via email/chat
- Use weak passwords
- Store credentials in plain text files
- Lose the keystore (you can't update your app without it)

---

## Next Steps

1. **Create keystore** (one-time)
2. **Sign unsigned APK** from GitHub Actions
3. **Optimize with zipalign**
4. **Verify signature**
5. **Upload to Play Store**

**For ongoing releases**, use the automated GitHub Actions workflow!
