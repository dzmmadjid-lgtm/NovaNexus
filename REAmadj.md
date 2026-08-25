name: Build Android APK

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v5
        with:
          distribution: temurin
          java-version: '17'

      - name: Set up Gradle
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '9.3.1'

      - name: Extract Android project
        run: |
          rm -rf project_files
          mkdir -p project_files
          unzip -o nova-nexus.zip -d project_files

          echo "======================================"
          echo "PROJECT EXTRACTED"
          echo "======================================"

          cd project_files
          ls -la

      - name: Create debug keystore
        working-directory: project_files
        run: |
          echo "======================================"
          echo "CREATING DEBUG KEYSTORE"
          echo "======================================"

          if [ -f "debug.keystore" ]; then
            echo "debug.keystore already exists."
          else
            keytool -genkeypair \
              -v \
              -keystore debug.keystore \
              -storepass android \
              -alias androiddebugkey \
              -keypass android \
              -keyalg RSA \
              -keysize 2048 \
              -validity 10000 \
              -dname "CN=Android Debug,O=Android,C=US"

            echo "debug.keystore created successfully."
          fi

          ls -lh debug.keystore

      - name: Check Gradle
        working-directory: project_files
        run: |
          echo "======================================"
          echo "GRADLE VERSION"
          echo "======================================"

          gradle --version

      - name: Build Debug APK
        working-directory: project_files
        run: |
          echo "======================================"
          echo "BUILDING NOVANEXUS APK"
          echo "======================================"

          gradle :app:assembleDebug \
            --stacktrace \
            --info \
            --no-daemon

      - name: Locate APK
        if: success()
        run: |
          echo "======================================"
          echo "APK FILES"
          echo "======================================"

          find project_files \
            -type f \
            -name "*.apk" \
            -print

      - name: Upload APK
        if: success()
        uses: actions/upload-artifact@v4
        with:
          name: NovaNexus-debug-apk
          path: project_files/app/build/outputs/apk/debug/*.apk
          if-no-files-found: error

      - name: Build completed
        if: success()
        run: |
          echo ""
          echo "======================================"
          echo "     NOVANEXUS BUILD SUCCESS"
          echo "======================================"
          echo ""

          find project_files/app/build/outputs/apk/debug \
            -type f \
            -name "*.apk" \
            -exec ls -lh {} \;
