# 🖥️ Build Machine Requirements - Redshift ODBC Driver

תיעוד מלא למה שצריך כדי לבנות את ה-ODBC driver (גרסת OPENID fix).

---

## 📋 Table of Contents
- [Hardware Requirements](#hardware-requirements)
- [Software Requirements](#software-requirements)
- [Dependencies to Build](#dependencies-to-build)
- [Setup Steps](#setup-steps)
- [Build Process](#build-process)
- [Troubleshooting](#troubleshooting)

---

## 💻 Hardware Requirements

### Minimum:
- **CPU:** 4 cores
- **RAM:** 8 GB
- **Disk:** 20 GB free space
- **OS:** Windows 10/11 or Windows Server 2019/2022

### Recommended:
- **CPU:** 8+ cores (משמעותית מאיץ את הבנייה)
- **RAM:** 16 GB (למניעת swapping בזמן compilation)
- **Disk:** 50 GB free space (SSD מומלץ)
- **OS:** Windows 11 Pro or Windows Server 2022

### Build Times (approximate):
| Hardware | Dependencies | Driver Build | Total |
|----------|--------------|--------------|-------|
| 4 cores, 8GB RAM, HDD | ~2 hours | ~30 min | **~2.5 hours** |
| 8 cores, 16GB RAM, SSD | ~1 hour | ~15 min | **~1.25 hours** |
| 16 cores, 32GB RAM, NVMe | ~30 min | ~10 min | **~40 min** |

---

## 🛠️ Software Requirements

### 1. Build Tools

#### Visual Studio 2022 Build Tools
```powershell
# Download from:
https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2022

# Required components:
- MSVC v143 - VS 2022 C++ x64/x86 build tools (Latest)
- Windows 10 SDK (10.0.20348.0 or later)
- C++ CMake tools for Windows
- C++ ATL for latest v143 build tools (x86 & x64)
```

**Installation:**
```powershell
# Silent install with required components
vs_BuildTools.exe --quiet --wait --norestart --nocache ^
  --add Microsoft.VisualStudio.Workload.VCTools ^
  --add Microsoft.VisualStudio.Component.VC.Tools.x86.x64 ^
  --add Microsoft.VisualStudio.Component.Windows10SDK.20348 ^
  --add Microsoft.VisualStudio.Component.VC.CMake.Project ^
  --add Microsoft.VisualStudio.Component.VC.ATL
```

#### CMake 3.20+
```powershell
# Download from:
https://cmake.org/download/

# Or install via Chocolatey:
choco install cmake --version=3.27.0 -y

# Verify:
cmake --version
# Should show: cmake version 3.27.0 or higher
```

#### WiX Toolset 3.14
```powershell
# Download from:
https://github.com/wixtoolset/wix3/releases/tag/wix3141rtm

# Direct link:
https://github.com/wixtoolset/wix3/releases/download/wix3141rtm/wix314.exe

# Install:
wix314.exe /install /quiet /norestart

# Verify:
candle.exe -?
light.exe -?
```

**Add to PATH:**
```powershell
$wixPath = "C:\Program Files (x86)\WiX Toolset v3.14\bin"
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";$wixPath", "Machine")
```

#### NASM (Netwide Assembler)
```powershell
# Required for OpenSSL build

# Via Chocolatey (easiest):
choco install nasm -y

# Or download:
https://www.nasm.us/pub/nasm/releasebuilds/2.16.01/win64/nasm-2.16.01-win64.zip

# Extract to: C:\Program Files\NASM
# Add to PATH: C:\Program Files\NASM

# Verify:
nasm --version
```

#### Strawberry Perl
```powershell
# Required for OpenSSL build

# Via Chocolatey:
choco install strawberryperl -y

# Or download:
https://strawberryperl.com/

# Verify:
perl --version
```

#### Git + Git LFS
```powershell
# Git:
choco install git -y

# Git LFS (required for binary files):
choco install git-lfs -y

# Initialize:
git lfs install

# Verify:
git --version
git lfs version
```

---

### 2. Optional but Recommended

#### Chocolatey Package Manager
```powershell
# Install (run as Administrator):
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Verify:
choco --version
```

#### 7-Zip (for extracting archives)
```powershell
choco install 7zip -y
```

---

## 📦 Dependencies to Build

אלו הספריות שצריך לבנות/להוריד **לפני** בניית ה-ODBC driver:

### 1. OpenSSL 1.1.1w

#### אופציה A: Pre-built (מומלץ, מהיר)
```powershell
# Download from Shining Light Productions:
$url = "https://slproweb.com/download/Win64OpenSSL-1_1_1w.exe"
$installer = "$env:TEMP\openssl-installer.exe"
Invoke-WebRequest -Uri $url -OutFile $installer

# Install silently:
Start-Process -FilePath $installer -ArgumentList "/VERYSILENT","/SP-","/SUPPRESSMSGBOXES","/DIR=C:\OpenSSL" -Wait -NoNewWindow

# Verify:
Test-Path "C:\OpenSSL\include\openssl\ssl.h"
Test-Path "C:\OpenSSL\lib\libssl.lib"
```

#### אופציה B: Build from source (איטי, אבל מלא שליטה)
```powershell
# Download:
git clone --depth 1 --branch OpenSSL_1_1_1w https://github.com/openssl/openssl.git C:\build\openssl-src

cd C:\build\openssl-src

# Configure:
perl Configure VC-WIN64A no-shared --prefix=C:\OpenSSL --openssldir=C:\OpenSSL\ssl

# Build:
nmake
nmake install

# Time: ~15-20 minutes
```

---

### 2. AWS SDK for C++ 1.11.743

```powershell
# Clone specific version:
git clone --depth 1 --branch 1.11.743 https://github.com/aws/aws-sdk-cpp.git C:\build\aws-sdk-cpp

# Create build directory:
mkdir C:\build\aws-sdk-cpp-build
cd C:\build\aws-sdk-cpp-build

# Configure CMake:
cmake -G "Visual Studio 17 2022" -A x64 ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_INSTALL_PREFIX=C:\aws-sdk-cpp ^
  -DBUILD_ONLY="core;redshift;sts;identity-management" ^
  -DENABLE_TESTING=OFF ^
  -DBUILD_SHARED_LIBS=OFF ^
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded ^
  C:\build\aws-sdk-cpp

# Build and install:
cmake --build . --config Release --target install -j 8

# Time: ~30-45 minutes (depends on cores)
```

**Critical flags:**
- `BUILD_ONLY` - רק המודולים שצריך (חוסך זמן!)
- `BUILD_SHARED_LIBS=OFF` - static linking
- `CMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded` - חייב להתאים ל-driver

---

### 3. c-ares 1.34.5

```powershell
# Clone:
git clone --depth 1 --branch v1.34.5 https://github.com/c-ares/c-ares.git C:\build\c-ares

# Build:
mkdir C:\build\c-ares-build
cd C:\build\c-ares-build

cmake -G "Visual Studio 17 2022" -A x64 ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_INSTALL_PREFIX=C:\c-ares ^
  -DCARES_STATIC=ON ^
  -DCARES_SHARED=OFF ^
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded ^
  C:\build\c-ares

cmake --build . --config Release --target install -j 8

# Time: ~5 minutes
```

---

### 4. GoogleTest 1.17.0

```powershell
# Clone:
git clone --depth 1 --branch v1.17.0 https://github.com/google/googletest.git C:\build\googletest

# Build:
mkdir C:\build\googletest-build
cd C:\build\googletest-build

cmake -G "Visual Studio 17 2022" -A x64 ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_INSTALL_PREFIX=C:\googletest ^
  -DBUILD_SHARED_LIBS=OFF ^
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded ^
  C:\build\googletest

cmake --build . --config Release --target install -j 8

# Time: ~5 minutes
```

---

## 🚀 Setup Steps

### צעד 1: התקנת כלים בסיסיים

```powershell
# Run as Administrator

# Install Chocolatey
Set-ExecutionPolicy Bypass -Scope Process -Force
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install all tools
choco install cmake --version=3.27.0 -y
choco install nasm -y
choco install strawberryperl -y
choco install git -y
choco install git-lfs -y
choco install 7zip -y

# Initialize Git LFS
git lfs install
```

### צעד 2: התקנת Visual Studio Build Tools

```powershell
# Download
Invoke-WebRequest -Uri "https://aka.ms/vs/17/release/vs_buildtools.exe" -OutFile "$env:TEMP\vs_buildtools.exe"

# Install with required components
Start-Process -FilePath "$env:TEMP\vs_buildtools.exe" -ArgumentList `
  "--quiet","--wait","--norestart","--nocache", `
  "--add","Microsoft.VisualStudio.Workload.VCTools", `
  "--add","Microsoft.VisualStudio.Component.VC.Tools.x86.x64", `
  "--add","Microsoft.VisualStudio.Component.Windows10SDK.20348", `
  "--add","Microsoft.VisualStudio.Component.VC.CMake.Project", `
  "--add","Microsoft.VisualStudio.Component.VC.ATL" `
  -Wait -NoNewWindow

# Time: ~10-15 minutes
```

### צעד 3: התקנת WiX Toolset

```powershell
Invoke-WebRequest -Uri "https://github.com/wixtoolset/wix3/releases/download/wix3141rtm/wix314.exe" -OutFile "$env:TEMP\wix314.exe"

Start-Process -FilePath "$env:TEMP\wix314.exe" -ArgumentList "/install","/quiet","/norestart" -Wait -NoNewWindow

# Add to PATH
$wixPath = "C:\Program Files (x86)\WiX Toolset v3.14\bin"
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";$wixPath", "Machine")
```

### צעד 4: בניית Dependencies

ראה סעיף "Dependencies to Build" למעלה.

**סקריפט אוטומטי:**

```powershell
# Create: C:\build-all-deps.ps1

# OpenSSL (pre-built)
Write-Host "Installing OpenSSL..."
$url = "https://slproweb.com/download/Win64OpenSSL-1_1_1w.exe"
$installer = "$env:TEMP\openssl-installer.exe"
Invoke-WebRequest -Uri $url -OutFile $installer
Start-Process -FilePath $installer -ArgumentList "/VERYSILENT","/SP-","/SUPPRESSMSGBOXES","/DIR=C:\OpenSSL" -Wait -NoNewWindow

# AWS SDK C++
Write-Host "Building AWS SDK C++..."
git clone --depth 1 --branch 1.11.743 https://github.com/aws/aws-sdk-cpp.git C:\build\aws-sdk-cpp
mkdir C:\build\aws-sdk-cpp-build -Force
cd C:\build\aws-sdk-cpp-build
cmake -G "Visual Studio 17 2022" -A x64 `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_INSTALL_PREFIX=C:\aws-sdk-cpp `
  -DBUILD_ONLY="core;redshift;sts;identity-management" `
  -DENABLE_TESTING=OFF `
  -DBUILD_SHARED_LIBS=OFF `
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded `
  C:\build\aws-sdk-cpp
cmake --build . --config Release --target install -j 8

# c-ares
Write-Host "Building c-ares..."
git clone --depth 1 --branch v1.34.5 https://github.com/c-ares/c-ares.git C:\build\c-ares
mkdir C:\build\c-ares-build -Force
cd C:\build\c-ares-build
cmake -G "Visual Studio 17 2022" -A x64 `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_INSTALL_PREFIX=C:\c-ares `
  -DCARES_STATIC=ON `
  -DCARES_SHARED=OFF `
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded `
  C:\build\c-ares
cmake --build . --config Release --target install -j 8

# GoogleTest
Write-Host "Building GoogleTest..."
git clone --depth 1 --branch v1.17.0 https://github.com/google/googletest.git C:\build\googletest
mkdir C:\build\googletest-build -Force
cd C:\build\googletest-build
cmake -G "Visual Studio 17 2022" -A x64 `
  -DCMAKE_BUILD_TYPE=Release `
  -DCMAKE_INSTALL_PREFIX=C:\googletest `
  -BUILD_SHARED_LIBS=OFF `
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded `
  C:\build\googletest
cmake --build . --config Release --target install -j 8

Write-Host "All dependencies built successfully!"
```

**הרצה:**
```powershell
powershell -ExecutionPolicy Bypass -File C:\build-all-deps.ps1
```

---

## 🔨 Build Process

### צעד 1: Clone הפרויקט

```powershell
# Clone with Git LFS
git clone https://github.com/ORELASH/amazon-redshift-odbc-driver.git C:\redshift-odbc
cd C:\redshift-odbc

# Checkout branch with OPENID fix
git checkout openid-only-fix

# Pull LFS files
git lfs pull
```

### צעד 2: צור exports_basic.bat

```bat
REM Create: C:\redshift-odbc\exports_basic.bat

set ENABLE_TESTING=0
set "RS_MULTI_DEPS_DIRS=C:\OpenSSL;C:\aws-sdk-cpp;C:\c-ares;C:\googletest"
set "RS_OPENSSL_DIR=C:\OpenSSL"
```

### צעד 3: הרץ Build

```powershell
cd C:\redshift-odbc

# Build with version
.\build64.bat --version=2.1.13.1 --build-type=Release

# Or minimal:
.\build64.bat
```

**Output files:**
- **MSI:** `C:\redshift-odbc\src\odbc\rsodbc\install\AmazonRedshiftODBC64_2.1.13.1.msi`
- **DLL:** `C:\redshift-odbc\cmake-build\install\lib\rsodbc64.dll`

---

## 🐛 Troubleshooting

### בעיה: "cmake.exe not found"
```powershell
# Add to PATH:
$cmakePath = "C:\Program Files\CMake\bin"
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";$cmakePath", "Machine")
```

### בעיה: "NASM not found" או "OpenSSL build fails"
```powershell
# Verify NASM in PATH:
where nasm
# Should show: C:\Program Files\NASM\nasm.exe

# If not, add:
$nasmPath = "C:\Program Files\NASM"
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";$nasmPath", "Machine")
```

### בעיה: "WiX candle.exe not found"
```powershell
# Check installation:
Test-Path "C:\Program Files (x86)\WiX Toolset v3.14\bin\candle.exe"

# Add to PATH:
$wixPath = "C:\Program Files (x86)\WiX Toolset v3.14\bin"
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";$wixPath", "Machine")

# Restart PowerShell!
```

### בעיה: "Visual Studio not found"
```powershell
# Verify installation:
$vsPath = "C:\Program Files\Microsoft Visual Studio\2022\BuildTools"
Test-Path "$vsPath\VC\Auxiliary\Build\vcvarsall.bat"

# If missing, reinstall Build Tools
```

### בעיה: "Cannot find OpenSSL headers"
```powershell
# Check exports_basic.bat:
Get-Content exports_basic.bat

# Should have:
# set "RS_OPENSSL_DIR=C:\OpenSSL"

# Verify files exist:
Test-Path "C:\OpenSSL\include\openssl\ssl.h"
Test-Path "C:\OpenSSL\lib\libssl.lib"
Test-Path "C:\OpenSSL\lib\libcrypto.lib"
```

### בעיה: "AWS SDK not found"
```powershell
# Verify installation:
Test-Path "C:\aws-sdk-cpp\include\aws\core\Aws.h"
Test-Path "C:\aws-sdk-cpp\lib\aws-cpp-sdk-core.lib"

# Check exports_basic.bat includes AWS SDK path
```

---

## 📊 Complete Setup Checklist

- [ ] Windows 10/11 or Server 2019/2022
- [ ] 8+ GB RAM, 50+ GB disk
- [ ] Visual Studio 2022 Build Tools installed
- [ ] CMake 3.27+ installed and in PATH
- [ ] WiX Toolset 3.14 installed and in PATH
- [ ] NASM installed and in PATH
- [ ] Strawberry Perl installed and in PATH
- [ ] Git + Git LFS installed
- [ ] OpenSSL 1.1.1w built/installed at C:\OpenSSL
- [ ] AWS SDK C++ 1.11.743 built at C:\aws-sdk-cpp
- [ ] c-ares 1.34.5 built at C:\c-ares
- [ ] GoogleTest 1.17.0 built at C:\googletest
- [ ] exports_basic.bat created with correct paths
- [ ] Repository cloned with Git LFS
- [ ] Branch openid-only-fix checked out

---

## ⏱️ Total Setup Time

| Stage | Time |
|-------|------|
| Install tools | 30 min |
| Install VS Build Tools | 15 min |
| Build OpenSSL (pre-built) | 5 min |
| Build AWS SDK C++ | 45 min |
| Build c-ares | 5 min |
| Build GoogleTest | 5 min |
| Clone & Build Driver | 20 min |
| **TOTAL** | **~2 hours** |

*(זמנים משוערים למכונה עם 8 cores, 16GB RAM, SSD)*

---

## 🎯 Quick Start Script

**הכל באחד - סקריפט מלא:**

```powershell
# Save as: C:\setup-build-machine.ps1

Write-Host "=== Redshift ODBC Build Machine Setup ===" -ForegroundColor Cyan

# 1. Install Chocolatey
Write-Host "`n[1/6] Installing Chocolatey..." -ForegroundColor Yellow
Set-ExecutionPolicy Bypass -Scope Process -Force
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# 2. Install tools
Write-Host "`n[2/6] Installing build tools..." -ForegroundColor Yellow
choco install cmake --version=3.27.0 -y
choco install nasm -y
choco install strawberryperl -y
choco install git -y
choco install git-lfs -y
git lfs install

# 3. Install VS Build Tools
Write-Host "`n[3/6] Installing Visual Studio Build Tools..." -ForegroundColor Yellow
Invoke-WebRequest -Uri "https://aka.ms/vs/17/release/vs_buildtools.exe" -OutFile "$env:TEMP\vs_buildtools.exe"
Start-Process -FilePath "$env:TEMP\vs_buildtools.exe" -ArgumentList `
  "--quiet","--wait","--norestart","--nocache", `
  "--add","Microsoft.VisualStudio.Workload.VCTools", `
  "--add","Microsoft.VisualStudio.Component.VC.Tools.x86.x64", `
  "--add","Microsoft.VisualStudio.Component.Windows10SDK.20348" `
  -Wait -NoNewWindow

# 4. Install WiX
Write-Host "`n[4/6] Installing WiX Toolset..." -ForegroundColor Yellow
Invoke-WebRequest -Uri "https://github.com/wixtoolset/wix3/releases/download/wix3141rtm/wix314.exe" -OutFile "$env:TEMP\wix314.exe"
Start-Process -FilePath "$env:TEMP\wix314.exe" -ArgumentList "/install","/quiet","/norestart" -Wait -NoNewWindow

# 5. Build dependencies
Write-Host "`n[5/6] Building dependencies..." -ForegroundColor Yellow
# [Copy the build-all-deps.ps1 content here]

# 6. Clone driver
Write-Host "`n[6/6] Cloning driver repository..." -ForegroundColor Yellow
git clone https://github.com/ORELASH/amazon-redshift-odbc-driver.git C:\redshift-odbc
cd C:\redshift-odbc
git checkout openid-only-fix
git lfs pull

Write-Host "`n=== Setup Complete! ===" -ForegroundColor Green
Write-Host "To build, run: cd C:\redshift-odbc && .\build64.bat" -ForegroundColor Green
```

---

**תאריך:** 2026-03-09
**גרסה:** 2.1.13.1 (OPENID Fix)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
