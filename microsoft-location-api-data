## What Windows sends to Microsoft's location service

I know google and others share exactly what, so i figured id make it public too

When `LocationProviderWiFi` is enabled (`AllowedInferenceType` = 2 or 3), 
Windows queries Microsoft's inference service with a list of observed WiFi 
BSSIDs. Based on reverse engineering of `CLocationAdapterWiFi` in 
`LocationFramework.dll`:

**Data collected per BSSID:**
- BSSID (MAC address)
- Signal strength (RSSI)
- Channel / frequency
- Timestamp of observation
- Fine Timing Measurement (FTM / 802.11mc) data when the AP supports it — 
  gives Microsoft actual radio time-of-flight distance to the AP, not just 
  signal strength. Gated by a Windows feature flag.

**How BSSIDs are collected:**
`CLocationAdapterWiFi::GetUsableCachedBssList` calls 
`WlanInternalGetNetworkBssListWithFTMData` — an undocumented Windows-internal 
variant of the WLAN API. This returns *every* BSSID the radio has seen in its 
scan cache, not just the one Windows is connected to. There is no per-BSSID 
filter at this layer.

**Quality gates before submission:**
`CLocationAdapterWiFi::TranslateAndValidateWlanCachedScanData` performs a few 
checks:
- BSSIDs with all-zero MAC addresses are dropped
- If total BSSID count is below a minimum threshold, the whole scan is rejected
- If too many BSSIDs are "stale" (older than a configurable age limit), the 
  whole scan is rejected

If the scan passes, it is submitted to Microsoft via the URL returned by 
`LSUtility::GetWebServiceURI` — typically `https://inference.location.live.net/`.

The specific wire format of the outbound request was not characterized in 
this writeup. Based on `c_wcszXSDN`/`c_wcszXSIN` strings referencing W3C XML 
Schema namespaces, the request is likely XML-based (possibly POX or SOAP).

## Provider priority

Windows Location Service aggregates results from multiple providers:
- `LocationProviderGnss` — GPS hardware
- `LocationProviderWiFi` — queries Microsoft's inference service
- `LocationProviderCell` — cell tower triangulation
- `LocationProviderIP` — IP-based geolocation
- `LocationProviderVenue` — indoor positioning

A Composite provider combines them. Empirically, when a GPS source is 
available, it outranks network-based providers — in testing, attaching a 
phone-based GPS source immediately overrode WiFi-based results without any 
configuration change. The full decision logic of the Composite provider was 
not characterized in this writeup.
