# CYW43 AP Follow-up Design Notes

## Context

PR #400 adds the minimal Ruby API needed to toggle CYW43 AP mode:

- `CYW43.enable_ap_mode(ssid, password, auth = CYW43::Auth::WPA2_AES_PSK)`
- `CYW43.disable_ap_mode`

That first step is intentionally small. It proves that PicoRuby can make a Pico W / Pico 2 W advertise an SSID from Ruby without introducing a larger wrapper yet.

## What We Observed

Manual check on Pico 2 W:

- `CYW43.init("JP")` succeeded
- `CYW43.enable_ap_mode("PICO-TIMER", "12345678")` made the SSID visible from a smartphone
- `CYW43.disable_ap_mode` made the SSID disappear

Observed limitation:

- a client could see the SSID and attempt to connect, but usable connectivity was not confirmed

At the moment PicoRuby does not expose enough AP-side state to tell whether the remaining issue is:

- association/auth timing
- AP-side IPv4 visibility
- DHCP behavior
- client-side retry / captive portal behavior

## Important SDK Detail

The Pico SDK already appears to do more for AP mode than PicoRuby currently exposes.

Relevant files:

- `src/rp2_common/pico_cyw43_arch/cyw43_arch.c`
- `lib/cyw43-driver/src/cyw43_lwip.c`
- `lib/cyw43-driver/src/cyw43_config.h`

Notable behavior in the SDK:

- `cyw43_arch_enable_ap_mode(...)` brings up `CYW43_ITF_AP`
- the CYW43 lwIP integration assigns default AP-side IPv4 settings
- the default AP address is `192.168.4.1`
- the default AP netmask is `255.255.255.0`
- the CYW43 lwIP layer calls `dhcp_server_init(...)` for the AP interface

This suggests that the next PicoRuby step should not start by reimplementing DHCP or inventing a high-level wrapper. The better next step is to expose enough AP-side observability to confirm what the SDK is already doing at runtime.

## Goal Of The Next PR

Make AP-side network state observable from Ruby.

This should help answer:

- what AP IPv4 address is active after `enable_ap_mode`
- whether the AP interface is really up
- whether PicoRuby can bind a TCP server on the AP side and serve a page

This is the minimum needed to move toward a PicoRuby version of `pcw_timer.py` without jumping straight to a large API surface.

## Recommended Scope For The Next PR

Recommended theme:

- expose AP-side status and IPv4 information
- update examples and docs to print AP-side network info
- do not add a MicroPython-style wrapper yet

Recommended Ruby API additions:

- `CYW43.ap_ipv4_address -> String?`
- `CYW43.ap_ipv4_netmask -> String?`
- `CYW43.ap_ipv4_gateway -> String?`

Optional if implementation cost stays small:

- `CYW43.ap_active? -> bool`

Recommended shell/doc follow-up:

- update `ifconfig` to show AP-side information when AP mode is enabled
- add a tiny AP status sample that prints the AP address after `enable_ap_mode`

## Why This Should Be Separate

This keeps the next PR focused on inspection rather than policy.

It avoids mixing together:

- AP enable/disable
- AP observability
- AP configuration
- DHCP behavior changes
- HTTP application logic

That separation should make it easier to review and easier to debug on hardware.

## Suggested Implementation Shape

Internally, PicoRuby should stop assuming that IPv4 helpers are STA-only.

Instead of only exposing:

- `CYW43.ipv4_address`
- `CYW43.ipv4_netmask`
- `CYW43.ipv4_gateway`

add internal helpers that accept a CYW43 interface id:

- `CYW43_ITF_STA`
- `CYW43_ITF_AP`

Then keep the current STA-oriented API stable while adding AP-specific entry points on top.

This preserves backward compatibility and avoids changing the meaning of existing methods.

## Explicit Non-Goals For The Next PR

Do not include these in the next PR unless a runtime blocker forces it:

- AP-side custom IP/netmask/gateway setters
- DHCP server reimplementation
- client list / station enumeration
- AP channel configuration
- `network.WLAN(network.AP_IF)` compatibility layer
- full `pcw_timer.py` port

## Likely PR After That

Once AP-side observability exists, the next follow-up can be chosen based on what hardware testing shows.

Two likely directions:

1. If SDK default AP IP + DHCP already works:
   add a minimal AP + `TCPServer` HTTP sample

2. If SDK default AP IP + DHCP is not sufficient in PicoRuby:
   add explicit AP-side IP configuration and any missing DHCP control

## Why This Matters For `pcw_timer.py`

`pcw_timer.py` needs more than just SSID broadcast.

It assumes:

- a client can join the AP
- the Pico has a reachable AP-side IP
- a browser can connect to an HTTP server running on the board

The current AP on/off API is a good first milestone, but the next milestone should be proving AP-side network visibility from Ruby before adding the application layer.
