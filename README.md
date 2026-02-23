# Xcode & macOS App Development Commands

Every command you need from first build to shipping a signed DMG. No fluff.

> **Note:** This is a work in progress and not finished yet. The full pipeline will be updated in the future.

## Best External Resources

- **[Rasmus Andersson's macOS distribution gist](https://gist.github.com/rsms/929c9c2fec231f0cf843a1a746a416f5)** — Code signing, notarization, quarantine, and distribution vehicles (DMG vs ZIP vs PKG). Written by the creator of Inter and former Spotify/Figma engineer.
- **[Dennis Babkin — "So You Want to Code-Sign macOS Binaries?"](https://dennisbabkin.com/blog/?t=how-to-get-certificate-code-sign-notarize-macos-binaries-outside-apple-app-store)** — End-to-end walkthrough with an automation script.
- **[Apple TN2339 — Building from the Command Line](https://developer.apple.com/library/archive/technotes/tn2339/_index.html)** — Official Apple reference.
- **[MacStadium — Making Sense of xcodebuild Arguments](https://macstadium.com/blog/making-sense-of-xcodebuild-arguments)** — Projects, workspaces, targets, and schemes explained.
- **[xcodebuild man page](https://keith.github.io/xcode-man-pages/xcodebuild.1.html)** — Full flag reference.

---

## 1. Project Discovery

```bash
# List all schemes, targets, and configurations
xcodebuild -list

# List available destinations (simulators, devices)
xcodebuild -showdestinations -scheme MyApp

# List installed SDKs
xcodebuild -showsdks

# Resolve Swift Package Manager dependencies
xcodebuild -resolvePackageDependencies
```

## 2. Build

```bash
# Debug build
xcodebuild -scheme MyApp -configuration Debug build

# Clean + build
xcodebuild -scheme MyApp clean build

# Universal binary (Apple Silicon + Intel)
xcodebuild -scheme MyApp \
  ARCHS="arm64 x86_64" \
  ONLY_ACTIVE_ARCH=NO \
  build

# Specify destination
xcodebuild -scheme MyApp \
  -destination 'platform=macOS' \
  build
```

## 3. Test

```bash
# Run all tests
xcodebuild -scheme MyApp test

# Run tests for a specific destination
xcodebuild -scheme MyApp \
  -destination 'platform=macOS' \
  test

# Run a specific test class
xcodebuild -scheme MyApp \
  -only-testing:MyAppTests/SomeTestClass \
  test
```

## 4. Archive

```bash
xcodebuild -scheme MyApp \
  -configuration Release \
  -archivePath ./build/MyApp.xcarchive \
  archive
```

If you omit `-archivePath`, the archive lands in `~/Library/Developer/Xcode/Archives/`.

## 5. Export

```bash
xcodebuild -exportArchive \
  -archivePath ./build/MyApp.xcarchive \
  -exportPath ./build/export \
  -exportOptionsPlist ExportOptions.plist
```

Minimal `ExportOptions.plist` for Developer ID distribution:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>
  <string>developer-id</string>
  <key>teamID</key>
  <string>YOUR_TEAM_ID</string>
</dict>
</plist>
```

## 6. Code Signing

### Find your identities

```bash
security find-identity -v -p codesigning
```

### Sign an app bundle (bottom-up)

Sign frameworks and helpers first, then the main binary, then the bundle:

```bash
# Sign embedded frameworks first
codesign -f -s "Developer ID Application: Your Name (TEAMID)" \
  -o runtime --timestamp \
  MyApp.app/Contents/Frameworks/SomeFramework.framework

# Sign the main app bundle
codesign -f -s "Developer ID Application: Your Name (TEAMID)" \
  -o runtime --timestamp \
  MyApp.app
```

### Ad-hoc signing (local only, no Apple account needed)

```bash
codesign -s "-" MyApp.app
```

### Verify a signature

```bash
codesign -dvv MyApp.app
```

### Key flags

| Flag | Purpose |
|------|---------|
| `-f` | Force re-sign |
| `-s "ID"` | Signing identity |
| `-o runtime` | Hardened Runtime (required for notarization) |
| `--timestamp` | Secure timestamp (survives certificate expiration) |
| `--entitlements file.plist` | Attach entitlements |

### Do NOT use `--deep`

It signs in an unpredictable order. Always sign bottom-up manually.

## 7. Notarization

### One-time setup: store credentials

```bash
xcrun notarytool store-credentials "my-profile" \
  --apple-id "you@email.com" \
  --team-id "XXXXXXXXXX" \
  --password "app-specific-password"
```

Generate the app-specific password at [appleid.apple.com](https://appleid.apple.com).

### Create archive for submission

Use `ditto`, not `zip`. Plain `zip` can break code signatures.

```bash
ditto -c -k --keepParent MyApp.app MyApp.zip
```

### Submit and wait

```bash
xcrun notarytool submit MyApp.zip \
  --keychain-profile "my-profile" \
  --wait
```

### Check the log if it fails

```bash
xcrun notarytool log SUBMISSION_UUID \
  --keychain-profile "my-profile" \
  log.json
```

### Staple the ticket

```bash
xcrun stapler staple MyApp.app
```

### Verify with Gatekeeper

```bash
spctl -a -vvv -t install MyApp.app
```

## 8. Package for Distribution

### DMG

```bash
# Using create-dmg (brew install create-dmg)
create-dmg \
  --volname "MyApp" \
  --window-size 600 400 \
  --icon-size 128 \
  --app-drop-link 400 200 \
  MyApp.dmg \
  ./build/export/

# Sign the DMG
codesign -s "Developer ID Application: Your Name (TEAMID)" \
  --timestamp MyApp.dmg

# Notarize the DMG
xcrun notarytool submit MyApp.dmg \
  --keychain-profile "my-profile" --wait
xcrun stapler staple MyApp.dmg
```

### PKG installer

```bash
# Build a component package
pkgbuild --root ./build/export \
  --install-location /Applications \
  --identifier com.yourname.myapp \
  --version 1.0 \
  MyApp-component.pkg

# Build a product archive (distribution package)
productbuild --package MyApp-component.pkg \
  --sign "Developer ID Installer: Your Name (TEAMID)" \
  MyApp.pkg
```

## 9. Quarantine & Troubleshooting

### Check quarantine status

```bash
xattr -l MyApp.app
```

Look for `com.apple.quarantine` in the output.

### Remove quarantine (for testing)

```bash
xattr -dr com.apple.quarantine MyApp.app
```

### Common Gatekeeper errors

| Error | Cause |
|-------|-------|
| "App is damaged and can't be opened" | Quarantine + missing/invalid signature |
| "Apple cannot check it for malicious software" | Signed but not notarized |
| "Unidentified developer" | Ad-hoc signed or no Developer ID |

## 10. Entitlements

### View entitlements on a signed binary

```bash
codesign -d --entitlements - MyApp.app
```

### Common entitlements

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <!-- App Sandbox -->
  <key>com.apple.security.app-sandbox</key><true/>

  <!-- Network access -->
  <key>com.apple.security.network.client</key><true/>

  <!-- File access -->
  <key>com.apple.security.files.user-selected.read-write</key><true/>

  <!-- Camera -->
  <key>com.apple.security.device.camera</key><true/>

  <!-- Microphone -->
  <key>com.apple.security.device.audio-input</key><true/>

  <!-- Hardened Runtime: allow unsigned libraries -->
  <key>com.apple.security.cs.disable-library-validation</key><true/>

  <!-- Hardened Runtime: allow JIT -->
  <key>com.apple.security.cs.allow-jit</key><true/>
</dict>
</plist>
```

## Quick Reference: The Full Pipeline

```
build → test → archive → export → sign → notarize → staple → verify → package → sign DMG → notarize DMG → staple DMG → ship
```

Or as commands:

```bash
# 1. Build & test
xcodebuild -scheme MyApp clean build
xcodebuild -scheme MyApp test

# 2. Archive
xcodebuild -scheme MyApp -configuration Release \
  -archivePath ./build/MyApp.xcarchive archive

# 3. Export
xcodebuild -exportArchive \
  -archivePath ./build/MyApp.xcarchive \
  -exportPath ./build/export \
  -exportOptionsPlist ExportOptions.plist

# 4. Sign (if not already signed during export)
codesign -f -s "Developer ID Application: ..." \
  -o runtime --timestamp ./build/export/MyApp.app

# 5. Notarize
ditto -c -k --keepParent ./build/export/MyApp.app MyApp.zip
xcrun notarytool submit MyApp.zip \
  --keychain-profile "my-profile" --wait
xcrun stapler staple ./build/export/MyApp.app

# 6. Verify
spctl -a -vvv -t install ./build/export/MyApp.app

# 7. Package as DMG
create-dmg --volname "MyApp" MyApp.dmg ./build/export/
codesign -s "Developer ID Application: ..." --timestamp MyApp.dmg
xcrun notarytool submit MyApp.dmg \
  --keychain-profile "my-profile" --wait
xcrun stapler staple MyApp.dmg

# Done. Ship it.
```
