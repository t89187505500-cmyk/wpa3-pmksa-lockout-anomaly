# WPA3-Personal Architecture Flaw: Dynamic PMKSA Lockout Anomaly

## Overview
This repository documents a cross-layer authorization logic flaw causing a transient Denial of Service (DoS) state on client stations. The anomaly has been consistently reproduced on MikroTik RouterOS v7 using hAP ax3 hardware, though the core mechanism is structurally tied to general standard fallback implementations.

## The Problem Formula
According to the IEEE 802.11 SAE (Simultaneous Authentication of Equals) standard, the Pairwise Master Key (PMK) is derived strictly from the pre-shared key (passphrase) and the SSID (acting as salt). 

However, the PMKID cache identifier calculation relies heavily on explicit session variables:
`PMKID = HMAC-SHA1-128(PMK, "PMK Name" | AA | SPA)`

Where:
* **AA** = Authenticator Address (Access Point BSSID)
* **SPA** = Supplicant Personal Address (Client Station MAC)

### The Lockout Loop Mechanics:
1. **State Reset:** The Access Point (AP) undergoes a factory reset or complete interface recreation. If the static `admin-mac` is not hardcoded, the OS dynamically generates a new dynamic BSSID. The text configuration (SSID and passphrase) remains identical.
2. **Cache Mismatch:** The legacy client station attempts a fast reconnect using its valid, stored PMKID cache. However, it calculates the hash based on the historical (pre-reset) BSSID.
3. **Silent Drop:** The router receives an invalid PMKID within the PMF (Protected Management Frames) context. Instead of triggering a clean fallback to a full, standard SAE authentication exchange, the system initiates a silent drop/lockout (mechanism unconfirmed, likely driver-level handling).
4. **Result:** The wireless interface enters a cyclic handshake timeout loop or falls into a `not running` state for the affected station.

## Evidence & Verification
The anomaly remains invisible at the default system logging levels. To expose the underlying state conflict, verbose wireless logging subroutines must be explicitly enabled.

**Debug Log Output:**
```text
wireless,debug,packet interface=wifi1: association rejected, no cached PMK
wireless,info wifi1: client [MAC] disconnected, handshake timeout
```

## Mitigation Protocols
1. **Station-Side:** Complete purging of the wireless network profile at the operating system level (forcing the Supplicant stack to clear its local PMKSA cache and rebuild key tables from scratch).
2. **Vendor-Side (Proposed Architecture Fix):** Implement an automatic flushing mechanism for dead dynamic PMKSA entries upon detecting an unaligned PMKID from a known SSID, or elevate the `no cached PMK` warning to the default global logging level to ensure procedural visibility.
