# Mobile Deployment

## Overview

Deploying a mobile app is more complex than deploying a web app. Native binaries must be compiled for each platform, signed with cryptographic certificates, and reviewed by Apple and Google before reaching users. Over-the-air (OTA) updates let you bypass this for JavaScript changes. EAS (Expo Application Services) wraps all of this into a coherent CLI and CI/CD workflow.

This chapter covers the complete deployment pipeline: signing setup, EAS Build for native binaries, EAS Submit for store submission, EAS Update for OTA releases, and CI/CD integration.

---

## Prerequisites

- [React Native Basics](react-native-basics.md) — Expo project setup
- Apple Developer account ($99/year) for iOS
- Google Play Console account ($25 one-time) for Android
- `eas-cli` installed globally: `npm install -g eas-cli`

---

## Core Concepts

### Build vs Update

```
Native Binary (EAS Build)              OTA Update (EAS Update)
──────────────────────────             ──────────────────────────
Contains: native code + JS bundle      Contains: only JS bundle + assets
Requires: Apple/Google review          Instant: downloaded on app launch
When: new native deps, permissions     When: JS/UI/logic changes only
Store: App Store / Play Store          Delivery: EAS Update CDN
Frequency: every few weeks/months      Frequency: every day if needed
Time: 15–45 minutes                    Time: seconds to publish
```

**Rule of thumb:** If you changed anything in `ios/` or `android/` directories, added a native module, or changed permissions — you need a native build. Pure JS/TypeScript changes → OTA update.

### Profiles

EAS uses build profiles defined in `eas.json` to configure different build types.

```json
// eas.json
{
  "cli": { "version": ">= 7.0.0", "appVersionSource": "remote" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": { "simulator": true },
      "android": { "buildType": "apk" }
    },
    "preview": {
      "distribution": "internal",
      "channel": "preview",
      "android": { "buildType": "apk" }
    },
    "production": {
      "autoIncrement": true,
      "channel": "production"
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "you@example.com",
        "ascAppId": "1234567890",
        "appleTeamId": "ABCDE12345"
      },
      "android": {
        "serviceAccountKeyPath": "./google-service-account.json",
        "track": "internal"
      }
    }
  }
}
```

---

## Hands-On Examples

### Initial EAS Setup

```bash
# 1. Install and authenticate
npm install -g eas-cli
eas login

# 2. Initialize EAS in your project
eas init
# This creates a project on expo.dev and adds projectId to app.json

# 3. Configure builds
eas build:configure
# Creates eas.json with default profiles

# 4. Set up credentials
eas credentials
# Interactive: generates/uploads signing certificates for iOS and Android
```

### App Configuration

```json
// app.json — complete production config
{
  "expo": {
    "name": "My App",
    "slug": "my-app",
    "version": "1.0.0",
    "runtimeVersion": {
      "policy": "appVersion"
    },
    "orientation": "portrait",
    "icon": "./assets/icon.png",
    "userInterfaceStyle": "automatic",
    "splash": {
      "image": "./assets/splash.png",
      "resizeMode": "contain",
      "backgroundColor": "#ffffff"
    },
    "ios": {
      "bundleIdentifier": "com.example.myapp",
      "buildNumber": "1",
      "supportsTablet": false,
      "infoPlist": {
        "NSCameraUsageDescription": "This app uses the camera to scan barcodes.",
        "NSLocationWhenInUseUsageDescription": "This app uses your location to show nearby items."
      }
    },
    "android": {
      "package": "com.example.myapp",
      "versionCode": 1,
      "adaptiveIcon": {
        "foregroundImage": "./assets/adaptive-icon.png",
        "backgroundColor": "#6366f1"
      },
      "permissions": [
        "android.permission.CAMERA",
        "android.permission.ACCESS_FINE_LOCATION"
      ]
    },
    "updates": {
      "url": "https://u.expo.dev/<your-project-id>",
      "enabled": true,
      "fallbackToCacheTimeout": 0,
      "checkAutomatically": "ON_LOAD"
    },
    "extra": {
      "eas": { "projectId": "your-project-id" }
    }
  }
}
```

### Running Builds

```bash
# Development build (for testing with expo-dev-client)
eas build --profile development --platform ios
eas build --profile development --platform android

# iOS Simulator build
eas build --profile development --platform ios --local
# Then: xcrun simctl install booted path/to/build.app

# Preview build (shareable APK/IPA for testers)
eas build --profile preview --platform all

# Production build
eas build --profile production --platform all

# Check build status
eas build:list

# Download the artifact
eas build:download --id <build-id>
```

### App Signing

**iOS signing** — EAS manages provisioning profiles and signing certificates automatically.

```bash
# Auto-manage credentials (recommended)
eas credentials --platform ios
# Select: "Manage all credentials with EAS"

# Manual — upload your own certificate
eas credentials --platform ios
# Select: "Upload a distribution certificate"
```

**Android signing** — upload keystore to EAS or let EAS generate one.

```bash
# Generate keystore via EAS (recommended — stores in EAS credentials)
eas credentials --platform android

# Import existing keystore
eas credentials --platform android
# Select: "Upload a keystore"
```

**Critical:** Store your Android keystore in a safe place. If you lose it and the app is published, you cannot release updates under the same package name.

### EAS Submit — Store Submission

```bash
# Submit latest build to App Store
eas submit --platform ios --latest

# Submit latest build to Google Play (internal testing track first)
eas submit --platform android --latest

# Submit specific build
eas submit --platform ios --id <build-id>
```

**App Store prerequisites:**
1. App created in App Store Connect with bundle ID matching `app.json`
2. Privacy policy URL
3. Screenshots for each required device size (use `npx expo screenshot`)
4. App metadata (description, keywords, category)

**Google Play prerequisites:**
1. App created in Play Console
2. Service account key JSON with "Release manager" permissions
3. First upload must be done manually via Play Console web UI (subsequent via API)

### EAS Update — OTA Updates

```bash
# Publish OTA update to production channel
eas update --channel production --message "Fix login bug"

# Publish to preview channel
eas update --channel preview --message "New profile screen"

# Rollback to previous update
eas update:rollback --channel production

# List updates
eas update:list
```

**How OTA updates work:**

```
App launch
  ↓
Check EAS Update CDN for new bundle matching:
  - runtimeVersion (native binary compatibility)
  - channel (production / preview / etc.)
  ↓
If update available:
  Download in background (or await on first launch)
  ↓
Apply on next app restart
```

**Runtime version** — determines compatibility between an OTA update and a native binary. When you change native code, bump the runtime version (or let `"policy": "appVersion"` do it automatically based on `version` in app.json).

```tsx
// Check for and apply updates programmatically
import * as Updates from 'expo-updates';

async function checkForUpdates() {
  if (!Updates.isEmbeddedLaunch) return; // skip in dev

  try {
    const update = await Updates.checkForUpdateAsync();
    if (update.isAvailable) {
      await Updates.fetchUpdateAsync();
      await Updates.reloadAsync(); // restart app with new bundle
    }
  } catch (error) {
    // Network error — continue with cached bundle
    console.warn('Update check failed:', error);
  }
}

// Show update prompt to user instead of silent reload
async function promptForUpdate() {
  const update = await Updates.checkForUpdateAsync();
  if (!update.isAvailable) return;

  Alert.alert(
    'Update Available',
    'A new version is ready. Restart now?',
    [
      { text: 'Later', style: 'cancel' },
      {
        text: 'Restart',
        onPress: async () => {
          await Updates.fetchUpdateAsync();
          await Updates.reloadAsync();
        },
      },
    ]
  );
}
```

---

## Common Patterns & Best Practices

### CI/CD Pipeline with GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # On PR: publish OTA update to preview channel
  preview:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - run: eas update --channel preview --message "PR ${{ github.event.number }}" --non-interactive

  # On main merge: OTA update to production
  update:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - run: eas update --channel production --message "Deploy ${{ github.sha }}" --non-interactive

  # On version tag: native build + store submission
  native-build:
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - run: eas build --platform all --profile production --non-interactive
      - run: eas submit --platform all --latest --non-interactive
```

### Environment Variables

```bash
# Set secret environment variables in EAS
eas secret:create --name API_URL --value "https://api.example.com" --scope project
eas secret:create --name SENTRY_DSN --value "https://xxx@sentry.io/123" --scope project

# List secrets
eas secret:list
```

```json
// eas.json — reference secrets in builds
{
  "build": {
    "production": {
      "env": {
        "APP_ENV": "production",
        "API_URL": "https://api.example.com"
      }
    },
    "preview": {
      "env": {
        "APP_ENV": "preview",
        "API_URL": "https://staging-api.example.com"
      }
    }
  }
}
```

```tsx
// Access in app code
const API_URL = process.env.EXPO_PUBLIC_API_URL; // EXPO_PUBLIC_ prefix = included in bundle
```

### Version Management

```bash
# Bump app version (updates app.json + eas.json autoIncrement)
eas build:version:set --version-code 42 --platform android
eas build:version:set --build-number 42 --platform ios

# Or use auto-increment in eas.json
{
  "build": {
    "production": {
      "autoIncrement": true  // increments versionCode/buildNumber per build
    }
  }
}
```

---

## Anti-Patterns to Avoid

**Using OTA updates to change native permissions.** OTA only delivers JS. If you add a new permission (camera, location), you must submit a native build through the store review process.

**Publishing OTA updates without testing on the preview channel first.** Always publish to a preview channel, test on real devices, then promote to production. One broken OTA update affects all your users instantly.

**Committing signing credentials to git.** Keystores, provisioning profiles, and p8 keys are secrets. Store them in EAS credentials (encrypted) or a secrets manager — never in your repository.

**Not setting a runtime version policy.** Without a runtime version, OTA updates will install on incompatible native binaries, causing crashes. Use `"policy": "appVersion"` to automatically link OTA compatibility to your native version.

**Skipping the internal test track on Google Play.** Before promoting to production, always release to "Internal testing" and test the actual store binary. Build artifacts from CI and store binaries can behave differently (ProGuard, signing differences).

**Blocking the main thread during updates.** `Updates.reloadAsync()` is destructive — it kills the app and restarts it. Never call it without user confirmation except on first launch.

---

## Debugging & Troubleshooting

### Build Failures

```bash
# View build logs
eas build:view --id <build-id>

# Run build locally for faster iteration
eas build --profile development --platform android --local
# Requires: Android Studio + NDK, or Xcode for iOS

# Validate eas.json
eas build --help
```

**Common iOS build failures:**
- `No profiles for bundle ID` — run `eas credentials` to register the bundle ID
- `Code signing error` — expired certificate; re-generate via `eas credentials`
- `Provisioning profile doesn't include device` — add UDID via `eas device:create`

**Common Android build failures:**
- `Keystore not found` — re-upload via `eas credentials`
- `Gradle build failed` — usually a native module incompatibility; check the module's compatibility with your Expo SDK version

### OTA Update Troubleshooting

```tsx
// Check current update info
import * as Updates from 'expo-updates';

console.log({
  updateId: Updates.updateId,
  channel: Updates.channel,
  runtimeVersion: Updates.runtimeVersion,
  isEmbeddedLaunch: Updates.isEmbeddedLaunch,
  manifest: Updates.manifest,
});
```

**Update not applying:** Check that `runtimeVersion` in the published update matches the installed binary. Mismatched runtime versions silently skip the update.

**"Update is too big":** EAS Update has a 50MB limit per update bundle. Use dynamic imports or split large features into native builds.

### Submission Rejections

**App Store:**
- 4.0 Design — not enough functionality; demo apps and shells are rejected
- 2.1 Performance crashes — test on oldest supported device (check your iOS deployment target)
- 5.1.1 Privacy — missing permission descriptions in `infoPlist`

**Google Play:**
- Policy violation — use the Policy Center to check; common: missing privacy policy link
- Target API level — must target the latest or previous Android API level

---

## Real-World Scenarios

### Phased OTA Rollout

```bash
# Publish to 10% of users on production channel
# (EAS Update does not natively support percentages — use feature flags)
# Pattern: deploy OTA to all, gate new features behind a flag from your server
```

### Hotfix Deployment

```bash
# Emergency fix: skip preview → straight to production OTA
git checkout -b hotfix/crash-on-login
# fix the bug
git commit -m "fix: null check on auth token"
eas update --channel production --message "Hotfix: null check on auth token" --non-interactive
```

### Multi-Environment App

```
Branch        → Channel     → Users
────────────────────────────────────────
main          → production  → All users
staging       → preview     → QA team
feature/*     → development → Dev devices
```

---

## Internal Distribution

Before submitting to stores, distribute builds to QA and stakeholders for testing.

**TestFlight (iOS):**

```bash
# Build and upload to TestFlight in one command
eas build --platform ios --profile preview
eas submit --platform ios --profile preview
# Testers receive an email invite; up to 10,000 external testers
# Internal group (up to 100 Apple accounts) gets immediate access
```

**Firebase App Distribution (Android + iOS):**

```bash
npm install --save-dev @react-native-firebase/app-distribution
```

```json
// eas.json — add a submit profile for Firebase App Distribution
{
  "submit": {
    "preview": {
      "android": {
        "serviceAccountKeyPath": "./firebase-service-account.json",
        "groups": ["qa-team", "stakeholders"],
        "releaseNotes": "Preview build for QA"
      },
      "ios": {
        "appleId": "your@email.com",
        "ascAppId": "1234567890",
        "groups": ["ios-testers"]
      }
    }
  }
}
```

```bash
eas build --platform android --profile preview
eas submit --platform android --profile preview
# Testers install the Firebase App Tester app and receive a notification
```

**Direct APK install (Android — fastest for internal devs):**

```bash
# Build an APK (not AAB) for sideloading
eas build --platform android --profile preview
# List recent builds and grab the download URL
eas build:list --platform android --limit 1
# Share the download link directly — testers enable "Install unknown apps"
```

Use TestFlight for iOS milestone testing, Firebase for cross-platform QA, and direct APK for developer devices.

---

## Staged Rollouts on Google Play

Release to a percentage of users, watch metrics, then expand. This limits blast radius if a build has a critical bug.

```bash
# EAS Submit supports setting initial rollout percentage
eas submit --platform android --profile production
# Then in Google Play Console → Production → Manage release → Edit rollout percentage
```

**Typical rollout ladder:**
1. 1% — smoke test with real users, watch crash-free rate
2. 5% — broader signal, check ANR rate
3. 20% — significant traffic, check p95 latency and ratings
4. 50% → 100% — if all metrics stable after 24-48h at each step

**Metrics to monitor between steps:**
- **Crash-free users:** must stay above 99.5% (Play Store threshold for demotion)
- **ANR rate:** must stay below 0.47% (Play Store threshold)
- **Rating delta:** watch for sudden drops in 1-star reviews
- **Firebase Performance:** app start time, network request latency

**Halt rollout immediately if metrics regress:**

```
Google Play Console → Production → Manage release → Halt rollout
```

This stops new installations of the current version — existing installs are unaffected. Fix the issue, push a new build, and start the ladder over.

**Apple App Store** does not support percentage rollouts for standard releases. Use feature flags (LaunchDarkly, Statsig, or a simple remote config) to gate new features independently of the release.

---

## OTA Update Rollback

If an EAS Update causes regressions, roll back without a store submission.

```bash
# See recent updates on the production branch
eas update:list --branch production --limit 10

# Roll back: republish a previous update group to the same branch
eas update:republish --group <previous-update-group-id> --branch production

# Alternative: publish a new update from a stable git commit
git checkout <stable-sha>
eas update --branch production --message "rollback: revert to stable"
git checkout -  # return to your working branch
```

**Detect OTA launches in-app** to monitor adoption and health:

```typescript
import * as Updates from 'expo-updates';
import analytics from '@segment/analytics-react-native';

export async function trackLaunchContext(): Promise<void> {
  if (!Updates.isEmbeddedLaunch) {
    // App launched from an OTA update (not the store binary)
    await analytics.track('app_ota_launch', {
      updateId: Updates.updateId,
      channel: Updates.channel,
    });
  }
}
```

**EAS Secrets — store credentials out of the repo:**

```bash
# Store sensitive files and strings in EAS (encrypted, per-project)
eas secret:create --scope project --name GOOGLE_SERVICES_JSON \
  --type file --value ./google-services.json

eas secret:create --scope project --name SENTRY_DSN \
  --type string --value "https://abc@sentry.io/123"

eas secret:create --scope project --name FIREBASE_APP_ID \
  --type string --value "1:1234567890:ios:abcdef"

# List all secrets
eas secret:list
```

```json
// eas.json — reference secrets as environment variables in builds
{
  "build": {
    "production": {
      "env": {
        "SENTRY_DSN": "$SENTRY_DSN",
        "FIREBASE_APP_ID": "$FIREBASE_APP_ID"
      }
    }
  }
}
```

Secrets are injected at build time by EAS Build workers — they never appear in your git history or build logs. Use `--scope account` for secrets shared across all your projects.

---

## Further Reading

- [EAS Build docs](https://docs.expo.dev/build/introduction/)
- [EAS Submit docs](https://docs.expo.dev/submit/introduction/)
- [EAS Update docs](https://docs.expo.dev/eas-update/introduction/)
- [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)
- [Google Play Policy Center](https://support.google.com/googleplay/android-developer/answer/9876714)
- [expo-updates API](https://docs.expo.dev/versions/latest/sdk/updates/)

---

## Summary

EAS Build compiles native binaries in the cloud — no local Xcode or Android Studio required. Three profiles cover the main use cases: `development` (dev client with hot reload), `preview` (shareable test build), `production` (store submission). EAS Submit automates App Store and Play Store submission. EAS Update publishes OTA updates that bypass store review — limited to JS and asset changes. CI/CD: trigger OTA updates on merge to main, native builds on version tags. Always set a `runtimeVersion` policy to prevent OTA updates from installing on incompatible native binaries. Never commit signing credentials to source control — use EAS credentials storage.
