# ⚡ Quick Build Guide - Build Locally (Windows)

מדריך מהיר לבניה עצמאית של Redshift ODBC Driver עם OPENID fix.

---

## 🎯 תמצית - מה צריך?

### חומרה מינימלית:
- Windows 10/11
- 8GB RAM
- 50GB פנויים
- 4+ CPU cores

### תוכנות (5 חובה):
1. **Visual Studio 2022 Build Tools** (או Visual Studio Community)
2. **CMake 3.27+**
3. **WiX Toolset 3.14**
4. **NASM** (עבור OpenSSL)
5. **Strawberry Perl** (עבור OpenSSL)

### Dependencies (4 ספריות):
1. **OpenSSL 1.1.1w**
2. **AWS SDK C++ 1.11.743**
3. **c-ares 1.34.5**
4. **GoogleTest 1.17.0**

---

## 🚀 התקנה מהירה (Copy-Paste)

### שלב 1: התקן Chocolatey (Package Manager)

```powershell
# Run PowerShell as Administrator
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

### שלב 2: התקן כלים בסיסיים

```powershell
# Still as Administrator
choco install cmake --version=3.27.0 -y
choco install nasm -y
choco install strawberryperl -y
choco install git -y
choco install git-lfs -y

# Initialize Git LFS
git lfs install
```

### שלב 3: התקן Visual Studio Build Tools

**אופציה A: קל יותר - Visual Studio Community (גדול, 5-10GB)**
```powershell
choco install visualstudio2022community -y
choco install visualstudio2022-workload-nativedesktop -y
```

**אופציה B: מינימלי - Build Tools בלבד (קטן יותר, 2-3GB)**
```powershell
# Download installer
Invoke-WebRequest -Uri "https://aka.ms/vs/17/release/vs_buildtools.exe" -OutFile "$env:TEMP\vs_buildtools.exe"

# Install
Start-Process -FilePath "$env:TEMP\vs_buildtools.exe" -ArgumentList `
  "--quiet","--wait","--norestart","--nocache", `
  "--add","Microsoft.VisualStudio.Workload.VCTools", `
  "--add","Microsoft.VisualStudio.Component.VC.Tools.x86.x64", `
  "--add","Microsoft.VisualStudio.Component.Windows10SDK.20348", `
  "--add","Microsoft.VisualStudio.Component.VC.CMake.Project", `
  "--add","Microsoft.VisualStudio.Component.VC.ATL" `
  -Wait -NoNewWindow
```

### שלב 4: התקן WiX Toolset

```powershell
# Download and install
Invoke-WebRequest -Uri "https://github.com/wixtoolset/wix3/releases/download/wix3141rtm/wix314.exe" -OutFile "$env:TEMP\wix314.exe"
Start-Process -FilePath "$env:TEMP\wix314.exe" -ArgumentList "/install","/quiet","/norestart" -Wait -NoNewWindow

# Add to PATH
$env:Path += ";C:\Program Files (x86)\WiX Toolset v3.14\bin"
[Environment]::SetEnvironmentVariable("Path", $env:Path, "Machine")
```

**⚠️ חשוב: סגור ופתח PowerShell חדש אחרי שלב 4!**

---

## 📦 שלב 5: בנה Dependencies

### 5.1: צור תיקיית עבודה

```powershell
# Create directories
New-Item -ItemType Directory -Force -Path "C:\odbc-build\deps"
New-Item -ItemType Directory -Force -Path "C:\odbc-build\build"
cd C:\odbc-build\build
```

### 5.2: התקן OpenSSL (מהיר - 5 דקות)

**אופציה A: Pre-built (מומלץ!)**
```powershell
# Download pre-built binary
$url = "https://slproweb.com/download/Win64OpenSSL-1_1_1w.exe"
$installer = "$env:TEMP\openssl-installer.exe"
Invoke-WebRequest -Uri $url -OutFile $installer

# Install silently
Start-Process -FilePath $installer -ArgumentList `
  "/VERYSILENT","/SP-","/SUPPRESSMSGBOXES","/DIR=C:\odbc-build\deps\openssl" `
  -Wait -NoNewWindow

# Verify
Test-Path "C:\odbc-build\deps\openssl\include\openssl\ssl.h"
```

**אופציה B: Build from source (איטי - 20 דקות)**
```powershell
# Clone
git clone --depth 1 --branch OpenSSL_1_1_1w https://github.com/openssl/openssl.git C:\odbc-build\build\openssl

cd C:\odbc-build\build\openssl

# Configure (need to run vcvarsall first)
"C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsall.bat" x64

perl Configure VC-WIN64A no-shared --prefix=C:\odbc-build\deps\openssl --openssldir=C:\odbc-build\deps\openssl\ssl

# Build
nmake
nmake install
```

### 5.3: בנה AWS SDK C++ (ארוך - 45 דקות)

```powershell
cd C:\odbc-build\build

# Clone specific version
git clone --depth 1 --branch 1.11.743 https://github.com/aws/aws-sdk-cpp.git aws-sdk-cpp

# Create build directory
mkdir aws-sdk-cpp-build
cd aws-sdk-cpp-build

# Configure
cmake -G "Visual Studio 17 2022" -A x64 `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_INSTALL_PREFIX="C:\odbc-build\deps\aws-sdk-cpp" `
  -DBUILD_ONLY="core;redshift;sts;identity-management" `
  -DENABLE_TESTING=OFF `
  -DBUILD_SHARED_LIBS=OFF `
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded `
  ..\aws-sdk-cpp

# Build (use all cores)
cmake --build . --config Release --target install -j 8

# Verify
Test-Path "C:\odbc-build\deps\aws-sdk-cpp\include\aws\core\Aws.h"
```

💡 **טיפ:** `-j 8` משתמש ב-8 cores. שנה לפי המעבד שלך.

### 5.4: בנה c-ares (מהיר - 5 דקות)

```powershell
cd C:\odbc-build\build

# Clone
git clone --depth 1 --branch v1.34.5 https://github.com/c-ares/c-ares.git c-ares

# Build
mkdir c-ares-build
cd c-ares-build

cmake -G "Visual Studio 17 2022" -A x64 `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_INSTALL_PREFIX="C:\odbc-build\deps\c-ares" `
  -DCARES_STATIC=ON `
  -DCARES_SHARED=OFF `
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded `
  ..\c-ares

cmake --build . --config Release --target install -j 8
```

### 5.5: בנה GoogleTest (מהיר - 5 דקות)

```powershell
cd C:\odbc-build\build

# Clone
git clone --depth 1 --branch v1.17.0 https://github.com/google/googletest.git googletest

# Build
mkdir googletest-build
cd googletest-build

cmake -G "Visual Studio 17 2022" -A x64 `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_INSTALL_PREFIX="C:\odbc-build\deps\googletest" `
  -DBUILD_SHARED_LIBS=OFF `
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded `
  ..\googletest

cmake --build . --config Release --target install -j 8
```

---

## 🏗️ שלב 6: בנה את ה-ODBC Driver

### 6.1: Clone הפרויקט

```powershell
cd C:\odbc-build

# Clone (with LFS!)
git clone https://github.com/ORELASH/amazon-redshift-odbc-driver.git driver
cd driver

# Checkout OPENID fix branch
git checkout openid-only-fix

# Pull LFS files
git lfs pull
```

### 6.2: צור exports_basic.bat

```powershell
# Create configuration file
@"
set ENABLE_TESTING=0
set "RS_MULTI_DEPS_DIRS=C:\odbc-build\deps"
set "RS_OPENSSL_DIR=C:\odbc-build\deps\openssl"
"@ | Out-File -FilePath "exports_basic.bat" -Encoding ASCII
```

### 6.3: בנה!

```powershell
# Build with version
.\build64.bat --version=2.1.13.1 --build-type=Release

# Or just:
.\build64.bat
```

**זמן משוער:** 15-20 דקות

---

## 📁 היכן הקבצים?

אחרי build מוצלח:

### MSI Installer:
```
C:\odbc-build\driver\src\odbc\rsodbc\install\AmazonRedshiftODBC64_2.1.13.1.msi
```

### DLL:
```
C:\odbc-build\driver\cmake-build\install\lib\rsodbc64.dll
```

### מבנה תיקיות:
```
C:\odbc-build\
├── deps\                    (Dependencies)
│   ├── openssl\
│   ├── aws-sdk-cpp\
│   ├── c-ares\
│   └── googletest\
├── build\                   (Build files - can delete after)
│   ├── aws-sdk-cpp\
│   ├── aws-sdk-cpp-build\
│   ├── c-ares\
│   ├── c-ares-build\
│   ├── googletest\
│   └── googletest-build\
└── driver\                  (ODBC Driver)
    ├── exports_basic.bat
    ├── build64.bat
    ├── src\
    ├── cmake-build\         (Build output)
    └── public\              (Artifacts)
```

---

## 💾 התקנה

### התקן את ה-MSI:

```powershell
# Install
cd C:\odbc-build\driver\src\odbc\rsodbc\install
msiexec /i AmazonRedshiftODBC64_2.1.13.1.msi

# Or double-click the MSI file
```

### בדוק התקנה:

```powershell
# Open ODBC Administrator
odbcad32.exe

# Should see: "Amazon Redshift (x64)" driver
```

---

## 🐛 פתרון בעיות נפוצות

### ❌ "cmake not found"
```powershell
# Add to PATH and restart PowerShell
$env:Path += ";C:\Program Files\CMake\bin"
[Environment]::SetEnvironmentVariable("Path", $env:Path, "Machine")
```

### ❌ "NASM not found"
```powershell
# Verify installation
where nasm

# If not found:
choco install nasm -y
# Then restart PowerShell
```

### ❌ "candle.exe not found" (WiX)
```powershell
# Add to PATH
$env:Path += ";C:\Program Files (x86)\WiX Toolset v3.14\bin"
[Environment]::SetEnvironmentVariable("Path", $env:Path, "Machine")
# Restart PowerShell!
```

### ❌ "Visual Studio not found"
```powershell
# Check installation
Test-Path "C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsall.bat"
Test-Path "C:\Program Files\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvarsall.bat"

# If both false, reinstall VS Build Tools
```

### ❌ Build fails with OpenSSL errors
```powershell
# Check exports_basic.bat
Get-Content exports_basic.bat

# Should show:
# set "RS_OPENSSL_DIR=C:\odbc-build\deps\openssl"

# Verify OpenSSL files exist
Test-Path "C:\odbc-build\deps\openssl\include\openssl\ssl.h"
Test-Path "C:\odbc-build\deps\openssl\lib\libssl.lib"
Test-Path "C:\odbc-build\deps\openssl\lib\libcrypto.lib"
```

### ❌ AWS SDK build fails
```powershell
# Check available disk space (need 10+ GB)
Get-PSDrive C

# Check if git clone completed
Test-Path "C:\odbc-build\build\aws-sdk-cpp\CMakeLists.txt"

# Try with fewer cores if running out of memory
cmake --build . --config Release --target install -j 2
```

### ❌ "Git LFS files missing"
```powershell
cd C:\odbc-build\driver

# Reinstall LFS
git lfs install

# Pull files
git lfs pull

# Verify
git lfs ls-files
```

---

## ⏱️ סיכום זמנים

| שלב | זמן |
|-----|-----|
| התקנת כלים | 20-30 דקות |
| OpenSSL (pre-built) | 5 דקות |
| AWS SDK C++ | 30-60 דקות |
| c-ares | 5 דקות |
| GoogleTest | 5 דקות |
| Build Driver | 15-20 דקות |
| **סה"כ** | **~1.5-2 שעות** |

*(זמנים עבור מכונה עם 8 cores, 16GB RAM, SSD)*

---

## 🎯 One-Liner Scripts

### סקריפט מלא - הכל באחד:

שמור כ-`C:\setup-and-build.ps1`:

```powershell
# ==============================================================================
# Redshift ODBC Driver - Complete Build Script
# ==============================================================================

$ErrorActionPreference = "Stop"

Write-Host "=== Redshift ODBC Driver Build - OPENID Fix ===" -ForegroundColor Cyan
Write-Host "Estimated time: 1.5-2 hours" -ForegroundColor Yellow

# Create workspace
Write-Host "`n[1/8] Creating workspace..." -ForegroundColor Yellow
New-Item -ItemType Directory -Force -Path "C:\odbc-build\deps" | Out-Null
New-Item -ItemType Directory -Force -Path "C:\odbc-build\build" | Out-Null

# Install Chocolatey
if (-not (Get-Command choco -ErrorAction SilentlyContinue)) {
    Write-Host "`n[2/8] Installing Chocolatey..." -ForegroundColor Yellow
    Set-ExecutionPolicy Bypass -Scope Process -Force
    iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
} else {
    Write-Host "`n[2/8] Chocolatey already installed" -ForegroundColor Green
}

# Install tools
Write-Host "`n[3/8] Installing build tools..." -ForegroundColor Yellow
choco install cmake --version=3.27.0 -y
choco install nasm -y
choco install strawberryperl -y
choco install git -y
choco install git-lfs -y
git lfs install

# OpenSSL
Write-Host "`n[4/8] Installing OpenSSL..." -ForegroundColor Yellow
$opensslUrl = "https://slproweb.com/download/Win64OpenSSL-1_1_1w.exe"
$installer = "$env:TEMP\openssl-installer.exe"
Invoke-WebRequest -Uri $opensslUrl -OutFile $installer
Start-Process -FilePath $installer -ArgumentList "/VERYSILENT","/SP-","/SUPPRESSMSGBOXES","/DIR=C:\odbc-build\deps\openssl" -Wait -NoNewWindow

# AWS SDK C++
Write-Host "`n[5/8] Building AWS SDK C++ (this takes ~45 min)..." -ForegroundColor Yellow
cd C:\odbc-build\build
git clone --depth 1 --branch 1.11.743 https://github.com/aws/aws-sdk-cpp.git aws-sdk-cpp
mkdir aws-sdk-cpp-build | Out-Null
cd aws-sdk-cpp-build
cmake -G "Visual Studio 17 2022" -A x64 -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX="C:\odbc-build\deps\aws-sdk-cpp" -DBUILD_ONLY="core;redshift;sts;identity-management" -DENABLE_TESTING=OFF -DBUILD_SHARED_LIBS=OFF -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded ..\aws-sdk-cpp
cmake --build . --config Release --target install -j 8

# c-ares
Write-Host "`n[6/8] Building c-ares..." -ForegroundColor Yellow
cd C:\odbc-build\build
git clone --depth 1 --branch v1.34.5 https://github.com/c-ares/c-ares.git c-ares
mkdir c-ares-build | Out-Null
cd c-ares-build
cmake -G "Visual Studio 17 2022" -A x64 -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX="C:\odbc-build\deps\c-ares" -DCARES_STATIC=ON -DCARES_SHARED=OFF -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded ..\c-ares
cmake --build . --config Release --target install -j 8

# GoogleTest
Write-Host "`n[7/8] Building GoogleTest..." -ForegroundColor Yellow
cd C:\odbc-build\build
git clone --depth 1 --branch v1.17.0 https://github.com/google/googletest.git googletest
mkdir googletest-build | Out-Null
cd googletest-build
cmake -G "Visual Studio 17 2022" -A x64 -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX="C:\odbc-build\deps\googletest" -DBUILD_SHARED_LIBS=OFF -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded ..\googletest
cmake --build . --config Release --target install -j 8

# Build Driver
Write-Host "`n[8/8] Building ODBC Driver..." -ForegroundColor Yellow
cd C:\odbc-build
git clone https://github.com/ORELASH/amazon-redshift-odbc-driver.git driver
cd driver
git checkout openid-only-fix
git lfs pull

@"
set ENABLE_TESTING=0
set "RS_MULTI_DEPS_DIRS=C:\odbc-build\deps"
set "RS_OPENSSL_DIR=C:\odbc-build\deps\openssl"
"@ | Out-File -FilePath "exports_basic.bat" -Encoding ASCII

.\build64.bat --version=2.1.13.1

Write-Host "`n=== BUILD COMPLETE! ===" -ForegroundColor Green
Write-Host "MSI Location: C:\odbc-build\driver\src\odbc\rsodbc\install\AmazonRedshiftODBC64_2.1.13.1.msi" -ForegroundColor Green
Write-Host "DLL Location: C:\odbc-build\driver\cmake-build\install\lib\rsodbc64.dll" -ForegroundColor Green
```

**הרצה:**
```powershell
# Run as Administrator
powershell -ExecutionPolicy Bypass -File C:\setup-and-build.ps1
```

---

## 🔄 Rebuild (בניה חוזרת)

אם כבר בנית dependencies פעם אחת:

```powershell
cd C:\odbc-build\driver

# Clean previous build
Remove-Item -Recurse -Force cmake-build -ErrorAction SilentlyContinue

# Rebuild (fast - only 15-20 min!)
.\build64.bat --version=2.1.13.1
```

---

## ✅ בדיקה שהתיקון עובד

אחרי התקנת ה-MSI:

1. **פתח ODBC Administrator:**
   ```powershell
   odbcad32.exe
   ```

2. **צור DSN חדש:**
   - System DSN → Add
   - בחר "Amazon Redshift (x64)"
   - הגדר Azure OAuth2

3. **אל תוסיף `openid` ידנית!** ה-driver יוסיף אוטומטית

4. **Test Connection**

5. **בדוק logs:**
   ```powershell
   # Logs location
   cd "$env:TEMP\Amazon Redshift ODBC Driver\logs"

   # Search for the fix
   Select-String -Path *.log -Pattern "Added 'openid' to scope"
   ```

אם רואה `"Added 'openid' to scope"` בלוגים - ✅ התיקון עובד!

---

**תאריך:** 2026-03-09
**גרסה:** 2.1.13.1 (OPENID Fix)
**Branch:** openid-only-fix

🤖 Generated with [Claude Code](https://claude.com/claude-code)
