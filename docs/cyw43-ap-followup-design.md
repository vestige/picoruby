# CYW43 AP Roadmap Toward `pcw_timer.py`

## Goal

The long-term goal is to make a PicoRuby implementation practical for the same class of application as [`pcw_timer.py`](https://github.com/vestige/ptc/blob/main/pcw_timer.py):

- the board starts a Wi-Fi AP
- the AP has a known reachable IP
- a browser can connect to an HTTP server running on the board
- Ruby can implement the timer application logic on top

The goal is not to clone the entire MicroPython `network.WLAN(network.AP_IF)` API in one step.
The goal is to reach the same practical use case through small, reviewable PRs.

## Where We Are Now

### Step 1: minimal AP on/off

Draft PR #400 adds the minimal Ruby API needed to toggle CYW43 AP mode:

- `CYW43.enable_ap_mode(ssid, password, auth = CYW43::Auth::WPA2_AES_PSK)`
- `CYW43.disable_ap_mode`

Manual check on Pico 2 W confirmed:

- `CYW43.init("JP")` succeeded
- `CYW43.enable_ap_mode("PICO-TIMER", "12345678")` made the SSID visible from a smartphone
- `CYW43.disable_ap_mode` made the SSID disappear

This proves that PicoRuby can make Pico W / Pico 2 W advertise an SSID from Ruby.

Observed limitation:

- a client could see the SSID and attempt to connect, but practical network use was not yet confirmed

### Step 2: AP-side IPv4 observability

The next small PR adds AP-side IPv4 getters:

- `CYW43.ap_ipv4_address`
- `CYW43.ap_ipv4_netmask`
- `CYW43.ap_ipv4_gateway`

It also updates:

- `ifconfig` to show separate `Station` and `Access Point` sections
- the AP sample to print the AP-side IP
- README and signatures

Implementation note:

- the internal IPv4 helpers now read from `CYW43_ITF_STA` and `CYW43_ITF_AP` explicitly
- AP-side getters return `nil` when the AP interface is not active, so stale values are not exposed after `disable_ap_mode`

This step does not try to change AP behavior.
It only makes AP-side network state observable from Ruby.

## Why Step 2 Matters

`pcw_timer.py` relies on the equivalent of `ap.ifconfig()[0]` before it starts its HTTP server.

Without AP-side observability, it is hard to answer:

- what IP the AP is using
- whether the AP-side interface is still up
- whether a client is failing because of Ruby code, AP config, or DHCP/connectivity behavior

AP-side IPv4 getters are therefore a good bridge between:

- "SSID is visible"
- "an HTTP app is reachable through the AP"

## Recommended Sequence From Here

### Keep the AP-side IPv4 getters PR small

Do not expand the current observability PR into a larger networking PR.

Reasons:

- it is already a coherent unit
- it gives a concrete hardware-checkable improvement
- adding policy/configuration changes now would make scope expand quickly

### Next small PR: experimental AP HTTP sample

After the IPv4 getter PR, the next small PR should be an experimental AP + HTTP sample.

Recommended scope:

- use `CYW43.enable_ap_mode`
- print `CYW43.ap_ipv4_address`
- start a tiny `TCPServer` on the board
- return a minimal HTTP response such as `Hello from PicoRuby AP mode`
- clearly document that the sample is experimental and intended for hardware confirmation

Recommended non-goals for that PR:

- no new CYW43 core API if it is not required
- no `network.WLAN(network.AP_IF)` wrapper
- no DHCP server reimplementation
- no AP-side custom IP setters
- no timer UI yet

Why this is the right next step:

- it is still small
- it is easy to try on hardware
- it directly tests whether PicoRuby can move from "AP is visible" to "HTTP is reachable"

## What To Decide After The Experimental HTTP Sample

The experimental AP HTTP sample should tell us which direction is needed next.

### If the sample works with the SDK defaults

Then the next follow-up can stay small and ergonomic:

- add AP status helpers if useful
- improve `ifconfig`
- consider a lightweight higher-level AP helper
- start porting the timer application in Ruby

### If the sample does not work reliably

Then the next follow-up should focus on missing AP-side control:

- AP status APIs
- AP-side `ifconfig`-like configuration
- DHCP/IP behavior investigation
- only then decide whether explicit DHCP control is needed

## Explicit Non-Goals For Now

These should stay out of the current AP-side IPv4 getter PR:

- full MicroPython `network.WLAN(network.AP_IF)` compatibility
- AP-side channel configuration
- connected client listing
- DHCP server reimplementation
- full `pcw_timer.py` port

Each of those can be revisited later if hardware testing shows they are necessary.

## Summary

The agreed path is:

1. keep the current AP-side IPv4 getters PR small
2. add a separate experimental AP HTTP sample PR
3. use that hardware result to choose whether the next work should be AP status, `ifconfig`/IP control, or DHCP-related support

This sequence keeps review scope small while still ensuring that every step can be exercised on real Pico hardware.
