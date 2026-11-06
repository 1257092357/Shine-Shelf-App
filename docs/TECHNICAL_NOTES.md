# Shine Shelf Link — DLNA architecture & compatibility notes

This document distinguishes protocol requirements and engineering recommendations from verified functionality in a specific release. It is not a claim that all items have already been implemented.

## 1. Protocol responsibilities

Shine Shelf Link acts as a **UPnP AV / DLNA MediaServer (DMS)**. A compatible control point or media player discovers the server, browses its `ContentDirectory`, and fetches media content using HTTP.

| Layer | Technology | Main responsibility |
|---|---|---|
| Discovery | **SSDP / UPnP** | Announce the media server and answer discovery requests (`M-SEARCH`) |
| Service description | Device description XML | Describe server identity, available services and control endpoints |
| Catalog | **ContentDirectory** | Enumerate shared containers/items with stable IDs and metadata |
| Capabilities | **ConnectionManager** | Describe supported transfer protocols and media types |
| Transport | **HTTP/1.1** | Serve bytes for local files and respond to client requests |
| Seeking | HTTP `Range` / `206 Partial Content` | Allow clients to request selected byte ranges |

The standard IPv4 SSDP multicast endpoint is `239.255.255.250:1900/UDP`. Clients often need to be on the same subnet, and routers must allow multicast traffic within the intended LAN.

## 2. Streaming stability checklist

- [ ] Bind discovery and HTTP endpoints to the correct active network interface(s); refresh announcements if the IP address changes.
- [ ] Reply to SSDP `M-SEARCH`; advertise `ssdp:alive` and send `ssdp:byebye` on clean shutdown where feasible.
- [ ] Use valid UPnP device/service XML and ContentDirectory responses; return correct URLs for accessible files.
- [ ] Implement **HTTP GET and HEAD**, correct `Content-Length` / `Content-Type`, and robust **byte-range requests** for seeking (`206`, `Content-Range`, `416`).
- [ ] Support large files without fully buffering them in memory.
- [ ] Handle filename Unicode, spaces, URL escaping, deep folders, unavailable paths, renamed/moved files, and duplicate filenames.
- [ ] Persist shared-folder configuration and validate paths/permissions on restart.
- [ ] Index folders incrementally and monitor changes without blocking the UI.
- [ ] Distinguish server availability, discovery success, and actual transfer throughput in the status UI.
- [ ] Test Ethernet + Wi-Fi, 2.4/5 GHz, router multicast isolation, VPN/firewall, network roaming, sleep/resume, and multiple client connections.
- [ ] Do not imply transcoding or decoding capabilities unless actually implemented; the receiving player determines compatible formats.
- [ ] Do not expose the server to the public Internet; ideally default to private LAN interfaces and user-selected share roots.

## 3. Network indicator semantics

**Two distinct speed metrics:**

- **Wi-Fi negotiated link rate (Mbps):** The adapter/driver's association rate, e.g. `1200 Mbps`. It is not application throughput.
- **Transfer rate (MB/s):** Data actually being served over HTTP during the sampling interval, e.g. `0.00 MB/s` while idle.

Suggested connectivity states:

| Color | Meaning | Recommended criteria |
|---|---|---|
| Green | Server ready | LAN connected, server listening, shared content accessible; optional discovery check passed |
| Yellow | Degraded | Weak/unstable link, discovery issues, or recurring stream retries |
| Red | Unavailable | No usable local network, server stopped, or files inaccessible |

Do **not** determine the color solely from RSSI: a strong Wi-Fi signal can coexist with blocked multicast, client isolation, or an unreachable HTTP service.

## 4. Windows implementation notes

- Allow the user to choose share roots; avoid exposing arbitrary paths outside them.
- Provide an optional startup setting and persist folder configuration.
- Show the actual local IP address of the active interface, not an unrelated VPN adapter.
- Verify the Windows Firewall rule permits application traffic on **trusted/private** networks.
- The Windows WLAN API exposes both link quality and link rates; these are different from measured HTTP throughput.

## 5. Android implementation notes

- Prefer the **Storage Access Framework** (`ACTION_OPEN_DOCUMENT_TREE`) for user-approved folder trees; retain URI grants as supported, rather than assuming unrestricted filesystem paths.
- Android 11+ restricts selection of some system/storage roots; test actual folder picker behavior.
- Long-running server activity is subject to Android background and foreground-service restrictions, and device-specific battery policies. Do not promise always-on background sharing or boot auto-start without testing across Android versions.
- Check local network permission requirements on newer Android versions and update the manifest and permission flow accordingly.
- Test Wi-Fi transitions, screen lock, Doze, app task removal, battery optimization, and content URI read performance.

## 6. Testing interoperability

A practical validation matrix should include:

- **Client:** Shine Shelf VR Player and at least one independent DLNA-compatible player.
- **Files:** small MP4, large MKV, JPEG, PNG, documents (if indexed), Unicode filenames, and nested directories.
- **Network:** IPv4 local Wi-Fi, wired LAN to Wi-Fi, different access points on same subnet, multicast-blocked guest Wi-Fi.
- **Actions:** server discover, browse, play, seek, pause, resume, folder toggle, rename/delete folder, disconnect/reconnect, restart service.
- **Observability:** capture SSDP traffic and HTTP responses without logging private filenames or full tokens unnecessarily.

## References (primary technical sources)

- [UPnP MediaServer specification](https://upnp.org/specs/av/UPnP-av-MediaServer-v1-Device.pdf)
- [UPnP Device Architecture](https://upnp.org/specs/arch/UPnP-arch-DeviceArchitecture-v1.1.pdf)
- [Microsoft DLNA transport requirements](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-dlnhnd/ef75437e-3cc0-44f6-a8d7-66ca62b0f979)
- [RFC 9110 — HTTP range requests](https://www.rfc-editor.org/rfc/rfc9110.html#section-14)
- [Windows WLAN real-time connection quality](https://learn.microsoft.com/en-us/windows/win32/api/wlanapi/ns-wlanapi-wlan_realtime_connection_quality)
- [Android Storage Access Framework](https://developer.android.com/training/data-storage/shared/documents-files)
- [Android foreground service restrictions](https://developer.android.com/develop/background-work/services/fgs/restrictions-bg-start)
- [Android local network permissions](https://developer.android.com/privacy-and-security/local-network-permission)
