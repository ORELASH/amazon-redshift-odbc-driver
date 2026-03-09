# 🏃 Self-Hosted Runner Setup Guide

מדריך לבניה על runner פרטי שלך במקום GitHub-hosted runners.

---

## 🎯 למה Self-Hosted Runner?

### יתרונות:
- ✅ **שליטה מלאה** על סביבת הבניה
- ✅ **Dependencies קבועים** - לא צריך להוריד כל פעם
- ✅ **מהיר יותר** - אין timeout של 6 שעות
- ✅ **חינמי** - אין הגבלת minutes
- ✅ **יותר זיכרון/CPU** אם צריך

### חסרונות:
- ❌ צריך לנהל ולתחזק בעצמך
- ❌ צריך לדאוג לאבטחה
- ❌ צריך מכונה שרצה 24/7 (או לפחות כשצריך builds)

---

## 📋 דרישות למכונת Runner

### חומרה מומלצת:
- **OS:** Windows 10/11 Pro or Windows Server 2019/2022
- **CPU:** 8+ cores
- **RAM:** 16+ GB
- **Disk:** 100+ GB free (SSD מומלץ)
- **Network:** חיבור יציב לאינטרנט

### תוכנה שצריך להתקין **לפני** שמוסיפים runner:

```powershell
# Run as Administrator

# 1. Install Chocolatey
Set-ExecutionPolicy Bypass -Scope Process -Force
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# 2. Install build tools
choco install cmake --version=3.27.0 -y
choco install nasm -y
choco install strawberryperl -y
choco install git -y
choco install git-lfs -y
git lfs install

# 3. Install Visual Studio Build Tools
Invoke-WebRequest -Uri "https://aka.ms/vs/17/release/vs_buildtools.exe" -OutFile "$env:TEMP\vs_buildtools.exe"
Start-Process -FilePath "$env:TEMP\vs_buildtools.exe" -ArgumentList `
  "--quiet","--wait","--norestart","--nocache", `
  "--add","Microsoft.VisualStudio.Workload.VCTools", `
  "--add","Microsoft.VisualStudio.Component.VC.Tools.x86.x64", `
  "--add","Microsoft.VisualStudio.Component.Windows10SDK.20348" `
  -Wait -NoNewWindow

# 4. Install WiX Toolset
Invoke-WebRequest -Uri "https://github.com/wixtoolset/wix3/releases/download/wix3141rtm/wix314.exe" -OutFile "$env:TEMP\wix314.exe"
Start-Process -FilePath "$env:TEMP\wix314.exe" -ArgumentList "/install","/quiet","/norestart" -Wait -NoNewWindow

# 5. Add to PATH
$wixPath = "C:\Program Files (x86)\WiX Toolset v3.14\bin"
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";$wixPath", "Machine")
```

---

## 🚀 התקנת Runner

### שלב 1: צור Runner ב-GitHub

1. **לך לrepo שלך:**
   ```
   https://github.com/ORELASH/amazon-redshift-odbc-driver
   ```

2. **Settings → Actions → Runners → New self-hosted runner**

3. **בחר:**
   - OS: Windows
   - Architecture: x64

4. **תראה הוראות להורדה והתקנה**

### שלב 2: הורד והתקן

GitHub יראה לך משהו כזה:

```powershell
# Download
mkdir actions-runner; cd actions-runner
Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-win-x64-2.311.0.zip -OutFile actions-runner-win-x64-2.311.0.zip
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::ExtractToDirectory("$PWD\actions-runner-win-x64-2.311.0.zip", "$PWD")

# Configure
.\config.cmd --url https://github.com/ORELASH/amazon-redshift-odbc-driver --token YOUR_TOKEN_HERE

# Run
.\run.cmd
```

### שלב 3: הגדר כ-Service (אופציונלי אבל מומלץ)

כדי שה-runner ירוץ אוטומטית כשהמכונה עולה:

```powershell
# Install as Windows Service (run as Administrator)
.\svc.sh install

# Start service
.\svc.sh start

# Check status
.\svc.sh status
```

---

## 📝 הוסף Labels ל-Runner

בזמן `config.cmd`, תשאל אותך על labels:

```
Enter any additional labels (ex. label-1,label-2): [press Enter to skip]
```

**הקלד:**
```
self-hosted,Windows,X64,redshift-build
```

**או אם יש לך dependencies מוכנות:**
```
self-hosted,Windows,X64,redshift-build,with-deps
```

---

## 🔧 שינוי ה-Workflow לשימוש ב-Runner שלך

### אופציה 1: Runner עם Dependencies מוכנות

אם הכנת dependencies פעם אחת על המכונה:

```yaml
# .github/workflows/build-openid-fix-self-hosted.yml
name: Build ODBC Driver - Self-Hosted

on:
  push:
    branches: [ openid-only-fix ]
  workflow_dispatch:

jobs:
  build-windows:
    # Use your runner instead of GitHub's
    runs-on: [self-hosted, Windows, X64, redshift-build]

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
      with:
        lfs: true
        fetch-depth: 0

    - name: Setup Git LFS
      run: git lfs pull

    - name: Read version
      id: version
      run: |
        $version = (Get-Content version.txt).Trim()
        echo "VERSION=$version" >> $env:GITHUB_OUTPUT
      shell: pwsh

    - name: Create exports_basic.bat
      run: |
        @"
        set ENABLE_TESTING=0
        set "RS_MULTI_DEPS_DIRS=C:\odbc-deps"
        set "RS_OPENSSL_DIR=C:\odbc-deps\openssl"
        "@ | Out-File -FilePath "exports_basic.bat" -Encoding ASCII
      shell: pwsh

    - name: Build ODBC Driver
      run: |
        .\build64.bat --dependencies-install-dir=C:\odbc-deps --version=${{ steps.version.outputs.VERSION }}
      shell: pwsh

    - name: Upload MSI Installer
      uses: actions/upload-artifact@v4
      with:
        name: redshift-odbc-openid-fix-v${{ steps.version.outputs.VERSION }}
        path: |
          src/odbc/rsodbc/install/*.msi
          src/odbc/rsodbc/install/*.msi.sha256
        retention-days: 90

    - name: Upload DLL
      uses: actions/upload-artifact@v4
      with:
        name: redshift-odbc-dll-v${{ steps.version.outputs.VERSION }}
        path: cmake-build/install/lib/*.dll
        retention-days: 30
```

**שים לב:**
- שינינו `runs-on` מ-`windows-2022` ל-`[self-hosted, Windows, X64, redshift-build]`
- הסרנו כל ההתקנות של tools (CMake, NASM, etc.)
- הנחנו שיש dependencies ב-`C:\odbc-deps`

### אופציה 2: Runner שבונה Dependencies כל פעם

אם רוצה לבנות dependencies בכל build (לא מומלץ, איטי):

```yaml
jobs:
  build-windows:
    runs-on: [self-hosted, Windows, X64]

    steps:
    # ... checkout ...

    - name: Build dependencies
      run: |
        # Same as current workflow - build OpenSSL, AWS SDK, etc.
      shell: pwsh

    - name: Build driver
      run: |
        .\build64.bat --dependencies-install-dir=D:\deps --version=...
      shell: pwsh
```

---

## 📦 הכנת Dependencies על ה-Runner (חד-פעמי)

אחרי שהתקנת את ה-runner, הכן dependencies:

### סקריפט להכנת Dependencies:

שמור כ-`C:\setup-odbc-deps.ps1` על ה-runner:

```powershell
$ErrorActionPreference = "Stop"

Write-Host "=== Setting up ODBC Dependencies ===" -ForegroundColor Cyan

# Create directory
New-Item -ItemType Directory -Force -Path "C:\odbc-deps" | Out-Null
New-Item -ItemType Directory -Force -Path "C:\odbc-build" | Out-Null

# OpenSSL
Write-Host "`n[1/4] Installing OpenSSL..." -ForegroundColor Yellow
$opensslUrl = "https://slproweb.com/download/Win64OpenSSL-1_1_1w.exe"
$installer = "$env:TEMP\openssl-installer.exe"
Invoke-WebRequest -Uri $opensslUrl -OutFile $installer
Start-Process -FilePath $installer -ArgumentList "/VERYSILENT","/SP-","/SUPPRESSMSGBOXES","/DIR=C:\odbc-deps\openssl" -Wait -NoNewWindow

# AWS SDK C++
Write-Host "`n[2/4] Building AWS SDK C++..." -ForegroundColor Yellow
cd C:\odbc-build
if (Test-Path aws-sdk-cpp) { Remove-Item -Recurse -Force aws-sdk-cpp }
git clone --depth 1 --branch 1.11.743 https://github.com/aws/aws-sdk-cpp.git aws-sdk-cpp

if (Test-Path aws-sdk-cpp-build) { Remove-Item -Recurse -Force aws-sdk-cpp-build }
mkdir aws-sdk-cpp-build | Out-Null
cd aws-sdk-cpp-build

cmake -G "Visual Studio 17 2022" -A x64 `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_INSTALL_PREFIX="C:\odbc-deps\aws-sdk-cpp" `
  -DBUILD_ONLY="core;redshift;sts;identity-management" `
  -DENABLE_TESTING=OFF `
  -DBUILD_SHARED_LIBS=OFF `
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded `
  ..\aws-sdk-cpp

cmake --build . --config Release --target install -j 8

# Copy source tree for CMake includes
Copy-Item -Path "C:\odbc-build\aws-sdk-cpp" -Destination "C:\odbc-deps\aws-sdk-cpp-src" -Recurse -Force

# c-ares
Write-Host "`n[3/4] Building c-ares..." -ForegroundColor Yellow
cd C:\odbc-build
if (Test-Path c-ares) { Remove-Item -Recurse -Force c-ares }
git clone --depth 1 --branch v1.34.5 https://github.com/c-ares/c-ares.git c-ares

if (Test-Path c-ares-build) { Remove-Item -Recurse -Force c-ares-build }
mkdir c-ares-build | Out-Null
cd c-ares-build

cmake -G "Visual Studio 17 2022" -A x64 `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_INSTALL_PREFIX="C:\odbc-deps\c-ares" `
  -DCARES_STATIC=ON `
  -DCARES_SHARED=OFF `
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded `
  ..\c-ares

cmake --build . --config Release --target install -j 8

# GoogleTest
Write-Host "`n[4/4] Building GoogleTest..." -ForegroundColor Yellow
cd C:\odbc-build
if (Test-Path googletest) { Remove-Item -Recurse -Force googletest }
git clone --depth 1 --branch v1.17.0 https://github.com/google/googletest.git googletest

if (Test-Path googletest-build) { Remove-Item -Recurse -Force googletest-build }
mkdir googletest-build | Out-Null
cd googletest-build

cmake -G "Visual Studio 17 2022" -A x64 `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_INSTALL_PREFIX="C:\odbc-deps\googletest" `
  -DBUILD_SHARED_LIBS=OFF `
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded `
  ..\googletest

cmake --build . --config Release --target install -j 8

Write-Host "`n=== Dependencies Ready! ===" -ForegroundColor Green
Write-Host "Location: C:\odbc-deps" -ForegroundColor Green
```

**הרץ על ה-runner:**
```powershell
powershell -ExecutionPolicy Bypass -File C:\setup-odbc-deps.ps1
```

**זמן:** ~1 שעה (פעם אחת!)

---

## 🔄 Workflow המלא לSelf-Hosted Runner

צור: `.github/workflows/build-self-hosted.yml`

```yaml
name: Build ODBC Driver - Self-Hosted Runner

on:
  push:
    branches: [ openid-only-fix ]
  workflow_dispatch:

jobs:
  build-windows:
    runs-on: [self-hosted, Windows, X64, redshift-build]

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
      with:
        lfs: true
        fetch-depth: 0

    - name: Setup Git LFS
      run: git lfs pull

    - name: Verify build tools
      run: |
        Write-Host "=== Verifying Build Environment ===" -ForegroundColor Cyan

        Write-Host "`nCMake:"
        cmake --version

        Write-Host "`nNASM:"
        nasm --version

        Write-Host "`nPerl:"
        perl --version | Select-String "This is perl"

        Write-Host "`nVisual Studio:"
        $vsPath = "C:\Program Files\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvarsall.bat"
        if (Test-Path $vsPath) {
          Write-Host "✓ Found: $vsPath" -ForegroundColor Green
        } else {
          Write-Host "✗ Not found: $vsPath" -ForegroundColor Red
          exit 1
        }

        Write-Host "`nWiX Toolset:"
        candle.exe -?

        Write-Host "`nDependencies:"
        Test-Path "C:\odbc-deps\openssl" | Out-String
        Test-Path "C:\odbc-deps\aws-sdk-cpp" | Out-String
        Test-Path "C:\odbc-deps\c-ares" | Out-String
        Test-Path "C:\odbc-deps\googletest" | Out-String
      shell: pwsh

    - name: Read version
      id: version
      run: |
        $version = (Get-Content version.txt).Trim()
        echo "VERSION=$version" >> $env:GITHUB_OUTPUT
      shell: pwsh

    - name: Create exports_basic.bat
      run: |
        @"
        set ENABLE_TESTING=0
        set "RS_MULTI_DEPS_DIRS=C:\odbc-deps"
        set "RS_OPENSSL_DIR=C:\odbc-deps\openssl"
        "@ | Out-File -FilePath "exports_basic.bat" -Encoding ASCII

        Write-Host "=== exports_basic.bat ===" -ForegroundColor Cyan
        Get-Content exports_basic.bat
      shell: pwsh

    - name: Clean previous build
      run: |
        if (Test-Path "cmake-build") {
          Remove-Item -Recurse -Force cmake-build
          Write-Host "✓ Cleaned cmake-build" -ForegroundColor Green
        }
        if (Test-Path "public") {
          Remove-Item -Recurse -Force public
          Write-Host "✓ Cleaned public" -ForegroundColor Green
        }
      shell: pwsh

    - name: Build ODBC Driver
      run: |
        Write-Host "=== Building ODBC Driver ===" -ForegroundColor Cyan
        .\build64.bat --dependencies-install-dir=C:\odbc-deps --version=${{ steps.version.outputs.VERSION }}
      shell: pwsh

    - name: Verify build outputs
      run: |
        Write-Host "=== Build Outputs ===" -ForegroundColor Cyan

        $msi = Get-ChildItem -Path "src\odbc\rsodbc\install" -Filter "*.msi" -Recurse -ErrorAction SilentlyContinue
        if ($msi) {
          Write-Host "✓ MSI: $($msi.FullName)" -ForegroundColor Green
          Write-Host "  Size: $([math]::Round($msi.Length/1MB, 2)) MB"
        } else {
          Write-Host "✗ MSI not found!" -ForegroundColor Red
          exit 1
        }

        $dll = Get-ChildItem -Path "cmake-build\install\lib" -Filter "*.dll" -Recurse -ErrorAction SilentlyContinue
        if ($dll) {
          Write-Host "✓ DLL: $($dll.FullName)" -ForegroundColor Green
          Write-Host "  Size: $([math]::Round($dll.Length/1MB, 2)) MB"
        } else {
          Write-Host "✗ DLL not found!" -ForegroundColor Red
        }
      shell: pwsh

    - name: Generate checksums
      run: |
        $msi = Get-ChildItem -Path "src\odbc\rsodbc\install" -Filter "*.msi" -Recurse | Select-Object -First 1
        if ($msi) {
          $hash = (Get-FileHash -Path $msi.FullName -Algorithm SHA256).Hash
          $hashFile = "$($msi.FullName).sha256"
          "$hash  $($msi.Name)" | Out-File -FilePath $hashFile -Encoding ASCII
          Write-Host "✓ SHA256: $hash" -ForegroundColor Green
        }
      shell: pwsh

    - name: Upload MSI Installer
      uses: actions/upload-artifact@v4
      with:
        name: redshift-odbc-openid-fix-v${{ steps.version.outputs.VERSION }}
        path: |
          src/odbc/rsodbc/install/*.msi
          src/odbc/rsodbc/install/*.msi.sha256
        retention-days: 90

    - name: Upload DLL
      uses: actions/upload-artifact@v4
      with:
        name: redshift-odbc-dll-v${{ steps.version.outputs.VERSION }}
        path: cmake-build/install/lib/*.dll
        retention-days: 30
        if-no-files-found: warn

  create-release:
    needs: build-windows
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/openid-only-fix'

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Read version
      id: version
      run: |
        VERSION=$(cat version.txt | tr -d ' ')
        echo "VERSION=$VERSION" >> $GITHUB_OUTPUT

    - name: Download artifacts
      uses: actions/download-artifact@v4
      with:
        name: redshift-odbc-openid-fix-v${{ steps.version.outputs.VERSION }}
        path: ./artifacts

    - name: Create Release
      uses: softprops/action-gh-release@v1
      with:
        tag_name: v${{ steps.version.outputs.VERSION }}-openid-fix
        name: ODBC Driver v${{ steps.version.outputs.VERSION }} - OPENID Fix (Self-Hosted)
        files: |
          artifacts/*.msi
          artifacts/*.sha256
        draft: false
        prerelease: false
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## ⚙️ בדיקת Runner

לאחר התקנה, בדוק שהrunner עובד:

```powershell
# On the runner machine
cd C:\actions-runner
.\run.cmd
```

תראה:
```
√ Connected to GitHub
√ Runner successfully added
√ Runner connection is good
Listening for Jobs
```

עכשיו תעשה push ו-workflow ירוץ על המכונה שלך!

---

## 🔒 אבטחה

### חשוב! Self-hosted runners הם פחות מאובטחים:

1. **אל תריץ על public repos** - רק private!
2. **השתמש ב-separate user** לא admin
3. **הגבל access** למכונה
4. **רשת מבודדת** אם אפשר
5. **עדכונים שוטפים** של Windows + tools

---

## 📊 יתרונות במספרים

| Aspect | GitHub-Hosted | Self-Hosted |
|--------|--------------|-------------|
| **Build Time (first)** | ~1.5 hours | ~15-20 min ⚡ |
| **Build Time (cached)** | ~30-40 min | ~15-20 min ⚡ |
| **Setup Time** | Every run | One-time |
| **Cost** | Free tier limited | Free (electricity) |
| **Control** | Limited | Full ✅ |
| **Reliability** | High | Depends on you |

---

## 🎯 סיכום

### צעדים:
1. ✅ הכן מכונה Windows עם build tools
2. ✅ התקן GitHub Actions runner
3. ✅ בנה dependencies פעם אחת
4. ✅ שנה workflow ל-`runs-on: [self-hosted, ...]`
5. ✅ Push → build רץ על המכונה שלך!

### תוצאה:
- **Build: 15-20 דקות** (במקום שעה!)
- **אין תלות ברשת** (dependencies מקומיים)
- **שליטה מלאה**

---

**תאריך:** 2026-03-09
**גרסה:** 2.1.13.1 (OPENID Fix)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
