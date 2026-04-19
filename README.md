# Disabling Microsoft's WiFi-Based Location Database on Windows

When Windows Location Services is polluted with incorrect data for your WiFi BSSIDs (e.g., your router was previously in a different location, and Microsoft's database hasn't self-corrected), browsers and apps using the Windows Location Service will report the wrong coordinates — even when Google's and Apple's databases have you correctly mapped.

This registry modification disables Windows' WiFi-based location inference, forcing fallback to IP-based geolocation. Reverse-engineered from `LocationFramework.dll` (`CLocationInferenceServices<ILocationWiFiBeaconInformation>::IsInferenceRequestAllowed`).

**Not officially documented by Microsoft. Use at your own risk.**

## Registry key

```
HKLM\SYSTEM\CurrentControlSet\Services\lfsvc\Components\LocationProviderWiFi\Settings
Value: AllowedInferenceType (DWORD)
Default: 3
```

## What each value does

The value is NOT a bitmask — it's an enum. Windows allows WiFi inference when `AllowedInferenceType` equals `3` (wildcard: allow for all engines) OR equals the specific value required for the current positioning engine (WiFi engine requires `2`).

| Value | WiFi lookup? | Effect |
|-------|--------------|--------|
| `0` | Blocked | Falls back to IP geolocation ✓ recommended |
| `1` | Blocked | Falls back to IP geolocation |
| `2` | **Allowed** | Queries Microsoft's BSSID database |
| `3` | **Allowed** | Default — queries Microsoft's BSSID database (wildcard) |
| `4` | Blocked | Falls back to IP geolocation |
| `5`+ | Blocked | Falls back to IP geolocation (not comprehensively tested) |

Any value that is **not 2 or 3** disables WiFi inference for this provider. `0` is the cleanest semantic choice.

## PowerShell — disable WiFi location inference

Run as Administrator:

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\lfsvc\Components\LocationProviderWiFi\Settings" -Name "AllowedInferenceType" -Value 0 -Type DWord
Restart-Service lfsvc -Force
```

## PowerShell — restore default

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\lfsvc\Components\LocationProviderWiFi\Settings" -Name "AllowedInferenceType" -Value 3 -Type DWord
Restart-Service lfsvc -Force
```

## PowerShell — verify current value

```powershell
(Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\lfsvc\Components\LocationProviderWiFi\Settings").AllowedInferenceType
```

## PowerShell — verify fix is working

Run after applying the change and restarting the service:

```powershell
Add-Type -AssemblyName System.Device
$w = New-Object System.Device.Location.GeoCoordinateWatcher
$w.Start(); Start-Sleep 5; $w.Position.Location; $w.Stop()
```

**Before fix:** coordinates of wherever Microsoft's polluted DB says you are.
**After fix:** IP-based coordinates (correct city/region, accuracy typically 100–500m instead of 30–100m).

## Tradeoffs

- Location accuracy on this machine degrades from WiFi-precision (~30–100m) to IP-precision (~100m–5km, varies by ISP)
- Other location providers (GPS, Cell, Bluetooth) are unaffected and continue working
- Your BSSIDs remain incorrectly mapped in Microsoft's database — this fix only prevents your machine from querying that data
- **Feature updates to Windows may reset this value.** Keep a script of the fix handy.
- Other devices on your network (other Windows PCs, phones using Microsoft's geolocation SDK) are not affected by this change — they would each need the fix applied individually

## Batch file version (alternative)

Save as `disable-wifi-location.bat`, right-click → Run as Administrator:

```batch
@echo off
reg add "HKLM\SYSTEM\CurrentControlSet\Services\lfsvc\Components\LocationProviderWiFi\Settings" /v AllowedInferenceType /t REG_DWORD /d 0 /f
net stop lfsvc
net start lfsvc
echo Done. WiFi-based location inference disabled.
pause
```

## Why this exists

Microsoft's WiFi geolocation database is crowdsourced from Windows devices reporting "I see these BSSIDs at these coordinates." Unlike Google and Apple's databases — which self-correct rapidly due to billions of GPS-equipped phones — Microsoft's database updates primarily come from Windows devices, which rarely have GPS hardware. When a BSSID is mapped to wrong coordinates (e.g., your router moved to a new address), there is no user-facing correction mechanism and no API to query/report errors. Polluted entries can persist for years.

This registry setting provides a workaround by disabling WiFi-based inference at the OS level, routing location queries through IP geolocation instead.

## Credits

Reverse-engineered via Ghidra analysis of `C:\Windows\System32\LocationFramework.dll` with Microsoft public PDB symbols. The gating function is `CLocationInferenceServices<T>::IsInferenceRequestAllowed`, located at ~`0x18005dad2` in Windows 11 24H2 builds (address will vary across builds).

---

**README suggestion for the repo:**

Title it something searchable like `windows-wifi-location-fix` or `fix-windows-location-service-wrong-location`. Add tags for `windows`, `geolocation`, `lfsvc`, `registry`, `privacy`. That way people hitting this exact problem via Google can actually find your repo.
