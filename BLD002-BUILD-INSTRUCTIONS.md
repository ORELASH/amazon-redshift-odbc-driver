# BLD002 Build Instructions - GitHub Actions

**Version:** 2.1.13.1-BLD002
**Commit:** 491ee69
**Status:** ✅ Pushed to GitHub - Build in progress

---

## 🚀 GitHub Actions Build Started!

ה-commit דחוף בהצלחה ל-GitHub וה-workflow אמור להתחיל אוטומטית.

### Build Details

**Repository:** https://github.com/ORELASH/amazon-redshift-odbc-driver
**Branch:** fix-azure-oauth-scope
**Commit:** 491ee69 - "fix: Test Connection UI freeze and timeout issues [BLD002-DEBUG]"

**Workflows שיופעלו:**
1. ✅ `build-msi.yml` - בונה MSI installer (Windows 2022)
2. ✅ `build-windows-driver.yml` - בונה driver + unit tests

---

## 📊 איך לבדוק סטטוס ה-Build

### דרך 1: GitHub Web UI (מומלץ)

1. **פתח את דף ה-Actions:**
   ```
   https://github.com/ORELASH/amazon-redshift-odbc-driver/actions
   ```

2. **חפש את ה-workflow האחרון:**
   - שם: "Build Redshift ODBC MSI" או "Build Windows ODBC Driver"
   - Branch: fix-azure-oauth-scope
   - Commit: "fix: Test Connection UI freeze and timeout issues [BLD002-DEBUG]"

3. **בדוק סטטוס:**
   - 🟡 **Yellow dot (in progress)** - Build רץ כרגע (~15-25 דקות)
   - ✅ **Green checkmark** - Build הצליח!
   - ❌ **Red X** - Build נכשל (בדוק logs)

---

### דרך 2: GitHub CLI (אם מותקן)

```bash
# התקנת gh CLI (אם צריך)
# Ubuntu/Debian:
sudo apt install gh

# התחברות
gh auth login

# בדיקת workflows
gh run list --branch fix-azure-oauth-scope --limit 5

# צפייה ב-workflow בזמן אמת
gh run watch

# הורדת logs
gh run view --log
```

---

## 📥 הורדת ה-MSI לאחר Build מוצלח

### שלב 1: מצא את ה-Artifact

1. לך ל-Actions page: https://github.com/ORELASH/amazon-redshift-odbc-driver/actions

2. לחץ על ה-workflow run האחרון (עם commit 491ee69)

3. גלול למטה ל-**"Artifacts"** section

4. תראה artifact בשם: **`redshift-odbc-msi`**

---

### שלב 2: הורד את ה-Artifact

**דרך Web UI:**
1. לחץ על `redshift-odbc-msi` artifact
2. הקובץ יורד כ-ZIP: `redshift-odbc-msi.zip`
3. חלץ את ה-ZIP

**תוכן ה-ZIP:**
```
redshift-odbc-msi/
├── RedshiftODBC-Community-v2.1.13.1-BLD002.msi
└── RedshiftODBC-Community-v2.1.13.1-BLD002.msi.sha256
```

---

### שלב 3: אמת SHA256 (מומלץ)

**Windows PowerShell:**
```powershell
# חשב SHA256
$hash = (Get-FileHash "RedshiftODBC-Community-v2.1.13.1-BLD002.msi" -Algorithm SHA256).Hash

# השווה עם הקובץ
$expected = Get-Content "RedshiftODBC-Community-v2.1.13.1-BLD002.msi.sha256"

if ($hash -eq $expected.Split()[0]) {
    Write-Host "✅ SHA256 verified!" -ForegroundColor Green
} else {
    Write-Host "❌ SHA256 mismatch!" -ForegroundColor Red
}
```

**Linux/macOS:**
```bash
# חשב SHA256
sha256sum RedshiftODBC-Community-v2.1.13.1-BLD002.msi

# השווה עם הקובץ
cat RedshiftODBC-Community-v2.1.13.1-BLD002.msi.sha256
```

---

## 🔍 מה ה-Build עושה?

### תהליך ה-Build (build-msi.yml)

**זמן משוער:** 15-25 דקות

**שלבים:**
1. **Setup (3-5 min):**
   - Windows 2022 runner
   - MSBuild, NASM, Perl
   - WiX Toolset for MSI creation

2. **Dependencies (8-12 min):**
   - vcpkg installation
   - OpenSSL compilation
   - AWS SDK C++ build
   - c-ares, GTest

3. **Driver Build (3-5 min):**
   - CMake configuration
   - C++ compilation
   - Linking with dependencies

4. **MSI Creation (1-2 min):**
   - WiX candle (compile .wxs)
   - WiX light (link MSI)
   - Embed dependencies

5. **Verification (<1 min):**
   - Calculate SHA256
   - Upload artifact

---

## 📦 פרטי ה-MSI שיבנה

**שם קובץ:**
```
RedshiftODBC-Community-v2.1.13.1-BLD002.msi
```

**תכונות:**
- Version: 2.1.13.1-BLD002
- Architecture: x64 (64-bit)
- Platform: Windows 7/8/10/11, Server 2012+
- Size: ~25-35 MB (משתנה לפי dependencies)

**כולל:**
- ✅ Test Connection fixes (window focus + timeout)
- ✅ Debug logging with [BLD002-DEBUG] markers
- ✅ Browser OAuth improvements
- ✅ All Community Edition features

---

## 🧪 בדיקת ה-MSI

### התקנה

1. **הרץ את ה-MSI:**
   ```
   RedshiftODBC-Community-v2.1.13.1-BLD002.msi
   ```

2. **וודא גרסה:**
   - פתח: Control Panel → Administrative Tools → ODBC Data Sources (64-bit)
   - Drivers tab
   - חפש: "Amazon Redshift ODBC Driver (x64)"
   - Version: **2.1.13.1-BLD002**

---

### הפעלת Logging

**Registry:**
```
HKEY_LOCAL_MACHINE\SOFTWARE\Amazon\Amazon Redshift ODBC Driver (x64)
```

**הוסף:**
- `LogLevel` (DWORD) = `4`
- `LogPath` (String) = `C:\temp\odbc_bld002.txt`

**או דרך PowerShell:**
```powershell
# Create directory
New-Item -Path "C:\temp" -ItemType Directory -Force

# Enable logging
reg add "HKLM\SOFTWARE\Amazon\Amazon Redshift ODBC Driver (x64)" /v LogLevel /t REG_DWORD /d 4 /f
reg add "HKLM\SOFTWARE\Amazon\Amazon Redshift ODBC Driver (x64)" /v LogPath /t REG_SZ /d "C:\temp\odbc_bld002.txt" /f

Write-Host "✅ Logging enabled: C:\temp\odbc_bld002.txt"
```

---

### בדיקת Test Connection

1. **צור DSN חדש:**
   - ODBC Data Sources → User DSN → Add
   - בחר: Amazon Redshift ODBC Driver (x64)
   - הגדר: Azure OAuth authentication

2. **לחץ "Test Connection":**
   - ✅ **צפוי:** דפדפן נפתח, אימות עובד, MessageBox מופיע בחזית
   - ✅ **צפוי:** אין timeout מוקדם (יש 120 שניות מלאות)
   - ✅ **צפוי:** אם סוגרים דפדפן → error ברור מיד

3. **בדוק את ה-log:**
   ```powershell
   Get-Content "C:\temp\odbc_bld002.txt" | Select-String "BLD002-DEBUG"
   ```

   **מה לחפש:**
   ```
   test_connect: login timeout set to 120 seconds [BLD002-DEBUG]
   test_connect: calling SQLDriverConnect, parent_hwnd=0x... [BLD002-DEBUG]
   test_connect: SQLDriverConnect returned rc=0 [BLD002-DEBUG]
   test_connect: SetForegroundWindow=1, BringWindowToTop=1 [BLD002-DEBUG]
   test_connect: MessageBox returned 1 [BLD002-DEBUG]
   ```

---

## 🐛 פתרון בעיות Build

### Build Failed - Dependency Download

**סימפטום:**
```
Failed to download package from GitHub/NuGet
```

**פתרון:**
- ה-workflow מנסה 3 פעמים אוטומטית
- אם נכשל: Run workflow שוב (Re-run all jobs)

---

### Build Failed - Compilation Error

**סימפטום:**
```
error C2xxx: compilation error
```

**פתרון:**
1. בדוק build logs:
   ```bash
   gh run view <run-id> --log > build.log
   ```

2. חפש את השגיאה הראשונה (לא האחרונה!)

3. אם זה בגלל השינויים שלנו:
   - בדוק syntax ב-setup.c
   - וודא שכל הסוגריים סגורים

---

### Artifact Not Found

**סימפטום:**
- Build הצליח אבל אין artifact

**פתרון:**
1. בדוק שלב "Find MSI file" ב-workflow
2. וודא ש-build64.bat הצליח
3. אם MSI לא נוצר - בדוק WiX logs

---

## 📞 תמיכה

### Logs למשלוח

אם יש בעיה, שלח:
1. ✅ GitHub Actions workflow URL
2. ✅ Build logs (Download from Actions → Logs)
3. ✅ ODBC log file (C:\temp\odbc_bld002.txt)
4. ✅ Windows version and ODBC Data Sources screenshot

---

## ⏱️ Timeline משוער

| שלב | זמן | סטטוס |
|------|------|--------|
| Push to GitHub | ✅ Done | Completed |
| Workflow trigger | +30 sec | Waiting |
| Setup environment | +3-5 min | Running |
| Download dependencies | +8-12 min | Running |
| Build driver | +3-5 min | Pending |
| Create MSI | +1-2 min | Pending |
| Upload artifact | +30 sec | Pending |
| **Total** | **15-25 min** | **In Progress** |

---

## 🎯 Next Steps

1. ⏳ **Wait for build** (~20 minutes)
   - Check: https://github.com/ORELASH/amazon-redshift-odbc-driver/actions

2. 📥 **Download MSI**
   - From Artifacts section

3. ✅ **Verify SHA256**
   - Compare hash values

4. 💿 **Install and test**
   - Run MSI
   - Enable logging
   - Test Connection with browser auth

5. 📊 **Review logs**
   - Check for [BLD002-DEBUG] markers
   - Verify timeout = 120 seconds
   - Verify window focus restoration

---

**Build initiated:** $(date)
**Expected completion:** ~$(date -d '+25 minutes' 2>/dev/null || echo "in 20-25 minutes")
**Commit:** 491ee69
**Files changed:** 11 files, +1384 lines

---

**בהצלחה! 🚀**

ה-MSI יהיה מוכן בעוד כ-20 דקות.
