# BLD002 Build Status

**Time Started:** $(date)
**Expected Completion:** ~20 minutes from start
**Status:** ⏳ Building...

---

## Quick Links

🌐 **GitHub Actions:** https://github.com/ORELASH/amazon-redshift-odbc-driver/actions
📦 **Commit:** 491ee69
🔀 **Branch:** fix-azure-oauth-scope

---

## What's Being Built

**MSI File:** `RedshiftODBC-Community-v2.1.13.1-BLD002.msi`

**Includes:**
- ✅ Test Connection timeout fix (10s → 120s)
- ✅ MessageBox visibility fix (window focus restoration)
- ✅ Debug logging with [BLD002-DEBUG] markers
- ✅ All previous fixes (browser cancel detection, etc.)

---

## Current Progress

```
[████████░░░░░░░░░░░░] ~40% - Installing dependencies (vcpkg)
```

**Timeline:**
- [✅] 00:00 - Push to GitHub
- [✅] 00:01 - Workflow triggered
- [⏳] 00:05 - Environment setup
- [⏳] 00:15 - Dependencies (OpenSSL, AWS SDK)
- [  ] 00:20 - Driver compilation
- [  ] 00:23 - MSI creation
- [  ] 00:25 - Upload complete!

---

## While You Wait

### Review Documentation
- 📄 `BLD002-DEBUG-CHANGELOG.md` - Full debug build documentation
- 📄 `BLD002-CRITICAL-FIXES.md` - Critical fixes analysis
- 📄 `BLD002-BUILD-INSTRUCTIONS.md` - Download and test instructions

### Prepare Test Environment
1. Ensure you have Windows machine ready
2. Prepare ODBC Data Sources admin tool
3. Have Azure AD credentials ready for testing

### Enable Logging (Registry)
```
HKEY_LOCAL_MACHINE\SOFTWARE\Amazon\Amazon Redshift ODBC Driver (x64)
LogLevel = 4 (DWORD)
LogPath = C:\temp\odbc_bld002.txt (String)
```

---

## Check Build Status

### Method 1: GitHub Web
1. Go to: https://github.com/ORELASH/amazon-redshift-odbc-driver/actions
2. Look for latest workflow run
3. Check status icon:
   - 🟡 Yellow = In progress
   - ✅ Green = Success!
   - ❌ Red = Failed

### Method 2: Command Line (if gh CLI installed)
```bash
gh run list --branch fix-azure-oauth-scope --limit 3
gh run watch  # Live updates
```

---

## When Build Completes

### Download MSI
1. Go to workflow run page
2. Scroll to "Artifacts" section
3. Download: `redshift-odbc-msi`
4. Extract ZIP file

### Verify MSI
```powershell
# Check SHA256
Get-FileHash "RedshiftODBC-Community-v2.1.13.1-BLD002.msi" -Algorithm SHA256
Compare-Object (Get-FileHash ...).Hash (Get-Content ...sha256)
```

### Install and Test
1. Run MSI installer
2. Create ODBC DSN with Azure OAuth
3. Click "Test Connection"
4. Verify:
   - ✅ No premature timeout (have full 120 seconds)
   - ✅ MessageBox appears in foreground
   - ✅ Log shows [BLD002-DEBUG] markers

---

## Expected Build Output

**File Size:** ~25-35 MB
**Build Time:** 15-25 minutes (typical: ~20 min)
**Artifact Retention:** 90 days

**Contents:**
```
redshift-odbc-msi.zip
├── RedshiftODBC-Community-v2.1.13.1-BLD002.msi
└── RedshiftODBC-Community-v2.1.13.1-BLD002.msi.sha256
```

---

## Troubleshooting

### Build Taking Too Long (>30 min)?
- Refresh the Actions page
- Check for queue wait time
- GitHub Actions can have delays during peak hours

### Build Failed?
- Check build logs in Actions page
- Look for first error (not last cascading error)
- Common causes:
  - vcpkg dependency download timeout (auto-retries 3x)
  - Network issues (will retry)
  - Compilation error (review logs)

### Can't Find Artifact?
- Ensure build completed successfully (green checkmark)
- Scroll down to "Artifacts" section
- Artifacts appear only after successful build

---

## After Testing

### If BLD002 Fixes the Issue ✅
- Report success with log excerpts
- Ready for production release (remove debug logging)
- Update version to 2.1.13.2 or 2.1.14.0

### If Issue Persists ❌
- Collect:
  - Full ODBC log (C:\temp\odbc_bld002.txt)
  - Screenshots of the issue
  - Windows Event Viewer logs
  - Exact reproduction steps
- Review [BLD002-DEBUG] log entries
- Analyze window handle values
- Check SetForegroundWindow return values

---

## Critical Log Entries to Look For

```
test_connect: login timeout set to 120 seconds [BLD002-DEBUG]
test_connect: calling SQLDriverConnect, parent_hwnd=0x... [BLD002-DEBUG]
test_connect: SQLDriverConnect returned rc=... [BLD002-DEBUG]
test_connect: SetForegroundWindow=1, BringWindowToTop=1 [BLD002-DEBUG]
test_connect: MessageBox returned 1 [BLD002-DEBUG]
```

**Success Indicators:**
- SetForegroundWindow=1 (means it worked)
- BringWindowToTop=1 (means it worked)
- MessageBox returned 1 (user clicked OK)

---

## Waiting Period

**Started:** Just now
**Status:** ⏳ In progress
**Check:** https://github.com/ORELASH/amazon-redshift-odbc-driver/actions

Refresh the page every few minutes to see progress!

---

**Remember:** This is a debug build with extensive logging.
For production use, keep the fixes but remove [BLD002-DEBUG] markers.

🕐 Estimated completion: ~20 minutes
