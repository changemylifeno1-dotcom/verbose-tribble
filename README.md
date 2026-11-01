name: Build ROHIT ONE Release APK

on:
  workflow_dispatch: # Direct Run button in GitHub
  push:
    branches: [ "main" ]

jobs:
  build:
    name: Build Android APK
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Java 17 (Temurin)
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Set up Flutter SDK
        uses: subosito/flutter-action@v2
        with:
          channel: 'stable'
          cache: true

      - name: Fetch Dependencies
        run: flutter pub get

      # 1. Decode Keystore from GitHub Secrets if available
      - name: Decode Keystore
        if: "${{ secrets.KEYSTORE_BASE64 != '' }}"
        env:
          KEYSTORE_BASE64: ${{ secrets.KEYSTORE_BASE64 }}
        run: |
          echo "$KEYSTORE_BASE64" | base64 --decode > android/app/upload-keystore.jks

      # 2. Configure key.properties automatically for signing
      - name: Configure key.properties
        if: "${{ secrets.KEYSTORE_BASE64 != '' }}"
        env:
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
        run: |
          cat <<EOF > android/key.properties
          storePassword=$KEYSTORE_PASSWORD
          keyPassword=$KEY_PASSWORD
          keyAlias=$KEY_ALIAS
          storeFile=upload-keystore.jks
          EOF

      # 3. Build Signed Release APK
      - name: Build signed release APK
        env:
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
        run: |
          flutter build apk --release --split-per-abi

      - name: Upload arm64-v8a Release APK (For Modern Phones)
        uses: actions/upload-artifact@v4
        with:
          name: ROHIT-ONE-arm64-v8a-release
          path: build/app/outputs/flutter-apk/app-arm64-v8a-release.apk

      - name: Upload Universal APK
        uses: actions/upload-artifact@v4
        with:
          name: ROHIT-ONE-Universal-release
          path: build/app/outputs/flutter-apk/app-release.apk
          
