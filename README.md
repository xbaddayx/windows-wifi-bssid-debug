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

# Microsoft Location Service Endpoints (Orion)

Reverse-engineered from `C:\Windows\System32\LocationFramework.dll`.

## The endpoints

| URL | Purpose |
|-----|---------|
| `https://inference.location.live.net/` | **Production inference** — WiFi BSSID/cell tower/Bluetooth beacon → coordinates lookup |
| `https://staging-inference.location.live.net/` | Staging environment for the above (Microsoft internal testing) |
| `https://partnernext-inference.location.live-int.com/` | Internal/partner test environment |
| `https://agps.location.live.net/` | Production AGPS — Assisted GPS ephemeris data |
| `https://partnernext-agps.location.live-int.com/` | AGPS internal/test environment |
| `https://dev.virtualearth.net/REST/...` | Bing Maps REST — reverse geocoding (coordinates → street address) |
| `https://login.microsoft.com` | Authentication |
| `http://inference.location.live.com` | Legacy HTTP inference URL (older builds) |
| `http://agps.location.live.com` | Legacy HTTP AGPS URL |

Internal Microsoft codename for the backend system: **Orion**.

## How the URL is picked

In `LSUtility::GetWebServiceURI(WS_REPORT_TYPE, WSConfig*, string*)`:

**Step 1 — report type decides which family:**

| Report type | Family |
|---|---|
| `1` or `2` | Inference |
| `3` or `4` | AGPS |
| other | Returns `0x80070057` (invalid argument) |

**Step 2 — environment selection:**

If a WIL feature flag is enabled, URL is picked from an array indexed by `WSConfig+0x68` (prod / staging / internal-test — array lives at `PTR_c_orionServerUrl_PROD_18016eeb0`).

If the feature flag is disabled (older path), `WSConfig::IsUsingIntEnvironment()` returns 0 for consumer installs → production URL.

**Step 3 — tile URL override:**

`GetOrionInferenceTileUrl()` is called. If it returns a non-empty string, that overrides the selected URL. Probably used for regional routing or server-side A/B tests.

**Step 4 — API path appended:**

`GetServiceEndpointAPI(reportType, url)` appends the specific API path (e.g. `/inferenceV21/Pox/Inference` or similar) to form the final URL.

**Net result for a normal consumer machine:** requests go to `https://inference.location.live.net/<api-path>`.

## How to find this yourself in Ghidra

**Setup:**
1. Copy `C:\Windows\System32\LocationFramework.dll` to a working folder (the file is locked while lfsvc is running)
2. Import into a Ghidra project
3. Load Microsoft's public PDB symbols (File → Load PDB File → configure Symbol Server URL to `https://msdl.microsoft.com/download/symbols`, local cache to `C:\symbols`, download and apply)
4. Run Auto Analyze with PDB analyzer enabled

**Find the endpoint strings:**
1. Search → For Strings (Ctrl+Shift+S)
2. Filter by `location.live` or `inference` or `https://`
3. You'll see entries with symbol names like `c_orionServerUrl_PROD`, `c_wcszInfere...`, `c_wcszAGPS...`

**Find the URL-picker function:**
1. Double-click the string `https://inference.location.live.net/` in the results
2. Jumps to its address in the Listing view
3. Right-click the label → References → Find References to
4. One of the referencing functions is `LSUtility::GetWebServiceURI` — that's the URL selector

**Optional deeper dives:**
- Navigate to `PTR_c_orionServerUrl_PROD_18016eeb0` (the URL array) to see the environment table in full
- Find references to `GetWebServiceURI` to find the HTTP senders that use these URLs
- Find references to the field `WSConfig + 0x68` to find where the environment gets configured

Addresses (like `18016eeb0`) will differ slightly across Windows builds. The function names stay stable because they come from Microsoft's public PDB symbols.
