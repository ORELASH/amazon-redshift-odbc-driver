# Fix Error 126 - Missing Dependencies

**Error:**
```
The setup routines for the Amazon Redshift ODBC Driver (x64) ODBC driver could not be loaded
due to system error code 126: The specified module could not be found.
(C:\Program Files\Amazon Redshift ODBC Driver x64\Drivers\rsodbcsetup.dll).
```

**Root Cause:** Missing Visual C++ Runtime libraries and/or OpenSSL DLLs

---

## ✅ Solution 1: Install Visual C++ Redistributable (מומלץ)

### הורד והתקן:

**Microsoft Visual C++ Redistributable 2015-2022 (x64):**
```
https://aka.ms/vs/17/release/vc_redist.x64.exe
```

### שלבי התקנה:

1. **הורד:** `vc_redist.x64.exe`

2. **הרץ כ-Administrator:**
   - Right-click → "Run as administrator"

3. **עקוב אחרי ה-wizard:**
   - Accept license
   - Install
   - Restart if needed

4. **בדוק שוב ב-ODBC Administrator:**
   - פתח: ODBC Data Sources (64-bit)
   - Drivers → Amazon Redshift ODBC Driver (x64)
   - אמור לעבוד ללא Error 126

---

## ✅ Solution 2: Check Existing Installations

אם יש לך Visual Studio או Visual C++ כבר מותקן:

### בדוק אילו גרסאות מותקנות:

**PowerShell:**
```powershell
Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\* |
  Where-Object {$_.DisplayName -like "*Visual C++*"} |
  Select-Object DisplayName, DisplayVersion
```

**צריך לראות:**
```
Microsoft Visual C++ 2015-2022 Redistributable (x64) - 14.xx.xxxxx
```

אם אין - התקן מ-Solution 1 למעלה.

---

## ✅ Solution 3: Install All VC++ Versions (אם 1+2 לא עבדו)

לפעמים צריך גרסאות מרובות:

### הורד את כולם:

1. **VC++ 2015-2022 (x64):**
   ```
   https://aka.ms/vs/17/release/vc_redist.x64.exe
   ```

2. **VC++ 2013 (x64):**
   ```
   https://aka.ms/highdpimfc2013x64enu
   ```

3. **VC++ 2012 (x64):**
   ```
   https://download.microsoft.com/download/1/6/B/16B06F60-3B20-4FF2-B699-5E9B7962F9AE/VSU_4/vcredist_x64.exe
   ```

### התקן לפי הסדר:
- 2012 → 2013 → 2015-2022

---

## 🔍 Debug: איזה DLL חסרה?

### שימוש ב-Dependency Walker:

1. **הורד Dependencies:**
   ```
   https://github.com/lucasg/Dependencies/releases
   ```

2. **הרץ Dependencies.exe**

3. **פתח את הקובץ:**
   ```
   C:\Program Files\Amazon Redshift ODBC Driver x64\Drivers\rsodbcsetup.dll
   ```

4. **חפש DLLs באדום (missing):**
   - `VCRUNTIME140.dll` → צריך VC++ 2015-2022
   - `MSVCP140.dll` → צריך VC++ 2015-2022
   - `libcrypto-3-x64.dll` → צריך OpenSSL (Solution 4)
   - `libssl-3-x64.dll` → צריך OpenSSL (Solution 4)
   - `aws-cpp-sdk-*.dll` → צריך AWS SDK (Solution 5)

---

## ✅ Solution 4: Missing OpenSSL (פחות נפוץ)

אם Dependencies מראה חסר `libcrypto` או `libssl`:

### Option A: Copy from System

אם יש לך OpenSSL מותקן (Git for Windows, Strawberry Perl, etc.):

**חפש:**
```powershell
Get-ChildItem -Path "C:\" -Filter "libcrypto-3-x64.dll" -Recurse -ErrorAction SilentlyContinue
Get-ChildItem -Path "C:\" -Filter "libssl-3-x64.dll" -Recurse -ErrorAction SilentlyContinue
```

**העתק לתיקיית Driver:**
```powershell
Copy-Item "path\to\libcrypto-3-x64.dll" "C:\Program Files\Amazon Redshift ODBC Driver x64\Drivers\"
Copy-Item "path\to\libssl-3-x64.dll" "C:\Program Files\Amazon Redshift ODBC Driver x64\Drivers\"
```

### Option B: Download OpenSSL

**Win64 OpenSSL v3.x:**
```
https://slproweb.com/products/Win32OpenSSL.html
```

הורד: **Win64 OpenSSL v3.x.x Light** (EXE installer)

---

## ✅ Solution 5: Missing AWS SDK DLLs (נדיר)

אם Dependencies מראה חסר `aws-cpp-sdk-*.dll`:

### זה לא אמור לקרות!

ה-MSI אמור לכלול את כל ה-AWS SDK DLLs.

**בדוק אם הקבצים קיימים:**
```powershell
Get-ChildItem "C:\Program Files\Amazon Redshift ODBC Driver x64\Drivers\" -Filter "aws-*.dll"
```

**אם אין:**
1. הסר את ה-driver (Uninstall)
2. התקן שוב את ה-MSI
3. אם עדיין חסר - בעיה ב-MSI packaging

---

## 🎯 Quick Fix (הכי מהיר)

**רוב המקרים נפתרים עם:**

```powershell
# 1. הורד VC++ Redistributable
Invoke-WebRequest -Uri "https://aka.ms/vs/17/release/vc_redist.x64.exe" -OutFile "$env:TEMP\vc_redist.x64.exe"

# 2. התקן
Start-Process -FilePath "$env:TEMP\vc_redist.x64.exe" -ArgumentList "/install","/quiet","/norestart" -Wait

# 3. בדוק שוב ODBC
odbcad32.exe
```

---

## 📊 Common Causes & Solutions

| Error Pattern | Cause | Solution |
|--------------|-------|----------|
| **Error 126** on `rsodbcsetup.dll` | Missing VC++ Runtime | Install VC++ 2015-2022 Redistributable |
| **Red DLLs** in Dependencies | Missing specific DLL | Copy from system or install package |
| **VCRUNTIME140.dll** missing | No VC++ 2015-2022 | Install VC++ Redistributable |
| **MSVCP140.dll** missing | No VC++ 2015-2022 | Install VC++ Redistributable |
| **libcrypto-3-x64.dll** missing | No OpenSSL 3.x | Install OpenSSL or copy DLL |
| **All DLLs missing** | Wrong ODBC bitness | Use 64-bit ODBC Administrator |

---

## ⚠️ Important Notes

### 1. Use Correct ODBC Administrator

**64-bit Driver → 64-bit ODBC Administrator:**
```
C:\Windows\System32\odbcad32.exe
```

**NOT 32-bit:**
```
C:\Windows\SysWOW64\odbcad32.exe  ❌ WRONG!
```

### 2. Restart After Installing VC++

לפעמים Windows צריך restart אחרי התקנת VC++ Redistributable.

### 3. Check Windows Updates

וודא ש-Windows מעודכן:
```
Settings → Windows Update → Check for updates
```

---

## 🐛 Still Not Working?

### Collect Debug Info:

**1. Check installed VC++ versions:**
```powershell
Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\* |
  Where-Object {$_.DisplayName -like "*Visual C++*"} |
  Format-Table DisplayName, DisplayVersion
```

**2. Check DLLs in driver folder:**
```powershell
Get-ChildItem "C:\Program Files\Amazon Redshift ODBC Driver x64\Drivers\" |
  Format-Table Name, Length
```

**3. Run Dependency Walker:**
- Download: https://github.com/lucasg/Dependencies/releases
- Open: `rsodbcsetup.dll`
- Screenshot missing DLLs (red)

**4. Check Event Viewer:**
```
Event Viewer → Windows Logs → Application
Look for errors from "ODBC" or "Redshift"
```

---

## 📞 If All Else Fails

### Manual DLL Copy (last resort):

1. **מצא מכונה עם Visual Studio מותקן**

2. **העתק DLLs:**
   ```
   From: C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Redist\MSVC\14.xx.xxxxx\x64\Microsoft.VC143.CRT\

   Files to copy:
   - vcruntime140.dll
   - vcruntime140_1.dll
   - msvcp140.dll
   - concrt140.dll
   ```

3. **הדבק ב:**
   ```
   C:\Program Files\Amazon Redshift ODBC Driver x64\Drivers\
   ```

4. **נסה שוב**

---

## ✅ Success Verification

**After installing VC++ Redistributable:**

1. **פתח ODBC Administrator (64-bit):**
   ```
   C:\Windows\System32\odbcad32.exe
   ```

2. **Drivers tab:**
   - צריך לראות: "Amazon Redshift ODBC Driver (x64)"
   - ✅ ללא שגיאה

3. **נסה ליצור DSN:**
   - User DSN → Add
   - בחר: Amazon Redshift ODBC Driver (x64)
   - ✅ אמור לפתוח את ה-configuration dialog

4. **אם זה עובד - הבעיה נפתרה!** 🎉

---

## 🔗 Download Links Summary

**Must Have:**
- [VC++ 2015-2022 Redistributable (x64)](https://aka.ms/vs/17/release/vc_redist.x64.exe)

**Optional (if needed):**
- [VC++ 2013 (x64)](https://aka.ms/highdpimfc2013x64enu)
- [VC++ 2012 (x64)](https://download.microsoft.com/download/1/6/B/16B06F60-3B20-4FF2-B699-5E9B7962F9AE/VSU_4/vcredist_x64.exe)
- [OpenSSL for Windows](https://slproweb.com/products/Win32OpenSSL.html)
- [Dependencies (DLL analyzer)](https://github.com/lucasg/Dependencies/releases)

---

**הפתרון המהיר ביותר:** התקן VC++ 2015-2022 Redistributable והכל יעבוד! ✅
