# OpenSSL 3.x Upgrade for v2.1.14.1

## Overview
Version 2.1.14.1 upgrades OpenSSL from 1.1.1 to 3.3.2 to support Azure OAuth2 authentication with AWS Identity Center.

## Changes Required

### For Windows Builds

The build system expects OpenSSL to be provided via the `RS_OPENSSL_DIR` environment variable or through `DEPENDENCIES_INSTALL_DIR`.

#### Option 1: Using Pre-built OpenSSL 3.3.2
1. Download OpenSSL 3.3.2 for Windows from: https://slproweb.com/products/Win32OpenSSL.html
   - Get: Win64 OpenSSL v3.3.2 (or latest 3.x)
   - Install to a location like `C:\OpenSSL-Win64`

2. Set environment variable before building:
   ```batch
   set DEPENDENCIES_INSTALL_DIR=C:\path\to\dependencies
   ```

3. Ensure OpenSSL is in the expected structure:
   ```
   DEPENDENCIES_INSTALL_DIR\
     └── openssl\
         └── Release\  (or Debug\ for debug builds)
             ├── include\
             │   └── openssl\
             │       └── *.h files
             └── lib\
                 ├── libssl.lib
                 └── libcrypto.lib
   ```

#### Option 2: Build OpenSSL 3.3.2 from Source
1. Download OpenSSL 3.3.2 source from: https://www.openssl.org/source/
2. Follow the build instructions in INSTALL.md
3. Install to the structure shown above

### For Linux/macOS Builds

The build expects OpenSSL via `RS_MULTI_DEPS_DIRS` or system-installed OpenSSL 3.x.

#### Option 1: System Package Manager
```bash
# Ubuntu/Debian
sudo apt-get install libssl-dev  # Ensure version 3.x

# RHEL/CentOS/Fedora
sudo yum install openssl-devel  # Ensure version 3.x

# macOS with Homebrew
brew install openssl@3
```

#### Option 2: Build from Source
```bash
# Download and extract OpenSSL 3.3.2
wget https://www.openssl.org/source/openssl-3.3.2.tar.gz
tar xzf openssl-3.3.2.tar.gz
cd openssl-3.3.2

# Configure and build
./config --prefix=/opt/openssl-3.3.2
make
make install

# Set environment variable
export RS_MULTI_DEPS_DIRS=/opt/openssl-3.3.2
```

## Building the Driver

### Windows
```batch
build64.bat --version=2.1.14.1 --dependencies-install-dir=C:\path\to\dependencies
```

### Linux/macOS
```bash
mkdir cmake-build && cd cmake-build
cmake -DRS_MULTI_DEPS_DIRS=/opt/openssl-3.3.2 \
      -DRS_ODBC_DIR=/usr/local/unixodbc \
      -DCMAKE_INSTALL_PREFIX=install \
      ..
make -j5
make install
```

## Verification

After building, verify the OpenSSL version:
```bash
# On Linux/macOS
strings librsodbc.so | grep "OpenSSL 3.3"

# On Windows
# Check the MSI package includes OpenSSL 3.3.2 DLLs
```

## Azure OAuth2 Support

With OpenSSL 3.3.2, the driver now properly supports:
- Azure AD OAuth2 authentication via BrowserIdcAuthPlugin
- Modern TLS 1.3 protocol
- Enhanced cryptographic algorithms required by Azure services

## Compatibility Notes

- OpenSSL 3.x is backward compatible with applications built against OpenSSL 1.1.1
- No code changes required in the driver beyond linking to OpenSSL 3.x
- The driver will automatically use OpenSSL 3.x APIs when available

## Known Issues

None at this time. The upgrade is a drop-in replacement.

## References

- OpenSSL 3.3.2 Release: https://www.openssl.org/source/
- OpenSSL 3.0 Migration Guide: https://www.openssl.org/docs/man3.0/man7/migration_guide.html
- Azure OAuth2 with AWS Identity Center: AWS IAM Identity Center documentation
