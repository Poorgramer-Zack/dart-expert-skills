---
name: "codemagic"
description: "Codemagic CI/CD pipeline setup for Flutter using YAML configuration. Use when automating builds, configuring code signing, or deploying to App Store and Google Play."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Flutter combined with Codemagic Best Practices (YAML approach)

## Goal
Implements Codemagic YAML configurations for Flutter deployment. If GitHub Actions + Fastlane is a "DIY assembled vehicle", then [Codemagic](https://codemagic.io/) is a "luxury sports car built specifically for Flutter." It natively understands Flutter, requires no extra tooling, and features native Apple M-series machines for extremely fast compilation.

## Instructions

The recommended approach is using `codemagic.yaml` for "Infrastructure as Code," maintaining it within the project repository alongside the source code.

### 1. Infrastructure and Cache Strategy
Place `codemagic.yaml` in the project root. A single file can contain multiple Workflows (e.g., executing iOS deployments and Android deployments separately).

**Best Caching Practice**: Caching is fundamental to saving CI costs. Be sure to cache Dart dependencies and native build tools.

```yaml
# 🌟 codemagic.yaml
workflows:
  android-release:
    name: Android Release Workflow
    instance_type: mac_mini_m1 # Specify M1 machine for acceleration
    environment:
      # 🌟 Specify the Flutter version
      flutter: stable 
    cache:
      cache_paths:
        - $HOME/.pub-cache # Dart packages
        - $HOME/.gradle/caches # Gradle dependencies
```

### 2. Environment Variables & Encrypted Keys
**NEVER** hardcode Keystore passwords or API Keys in the YAML file. Codemagic provides highly secure UI dashboards to create variable groups, encrypt them, and consume them via scripts.

```yaml
    environment:
      flutter: stable
      groups:
        # Pre-configured group names on the Codemagic UI
        - keystore_credentials # Contains $KEY_PASSWORD, $ALIAS_PASSWORD
        - google_play_credentials # Contains the JSON key path for the Google Service Account
```

### 3. Analyze, Generate & Test
Before building, utilize the `scripts` block to execute commands sequentially.

```yaml
    scripts:
      - name: ⬇️ Fetch Dependencies
        script: flutter packages pub get
        
      - name: ⚙️ Code Generation (build_runner)
        script: dart run build_runner build --delete-conflicting-outputs
        
      - name: 🚨 Static Analysis
        script: flutter analyze
        
      - name: 🧪 Unit & Widget Tests
        script: flutter test
        # 🌟 Codemagic automatically parses the test results and displays them on a sleek Web Dashboard
        test_report: build/test-results/flutter.json 
```

### 4. Build Bundle / IPA
Codemagic's greatest strength lies in condensing complex Code Signing flows into extremely minimalist declarative syntax.

#### 4.1 Android Building (with Keystore)

```yaml
      - name: 🔨 Build Android App Bundle
        script: |
          # Generate a key.properties for android/app/build.gradle to read
          echo "storePassword=$KEYSTORE_PASSWORD" >> android/key.properties
          echo "keyPassword=$KEY_PASSWORD" >> android/key.properties
          echo "keyAlias=$KEY_ALIAS" >> android/key.properties
          echo "storeFile=$KEYSTORE_PATH" >> android/key.properties
          
          # 🌟 The build action
          flutter build appbundle --release
```

#### 4.2 iOS Code Signing
You typically do not need to manually configure Fastlane Match. Upload your `App Store Connect API Key` via Codemagic's Web Dashboard, and it automatically handles certificate fetching:

```yaml
    environment: # Declare before Workflow logic: Enable Auto Signing
      ios_signing:
        distribution_type: app_store
        bundle_identifier: com.yourcompany.app

    scripts:
      - name: 🍏 Build iOS IPA
        script: flutter build ipa --release
```

### 5. Push to Store (Publishing)
Once the build concludes, declare the target artifacts, and automatically push directly to the stores utilizing built-in modules—no upload scripting required!

```yaml
    artifacts:
      - build/app/outputs/bundle/release/**/*.aab
      - build/ios/ipa/*.ipa

    publishing:
      # Auto-email the team
      email:
        recipients:
          - devteam@company.com
          
      # 🌟 Publish to Google Play (Internal Track)
      google_play:
        credentials: $GCP_SERVICE_ACCOUNT_CREDENTIALS # The JSON key variable
        track: internal
        
      # 🌟 Publish to App Store Connect / TestFlight
      app_store_connect:
        auth: integration # Corresponds to the integrated Apple account from the Dashboard
        submit_to_testflight: true
```

## Constraints
*   **Target Audience**: Utilize `codemagic.yaml` when working with pure Flutter teams possessing sufficient budget constraints requiring extensive pipeline speeds (M-chip), avoiding complex Fastlane configurations. Wait for further explicit overrides if needed.
