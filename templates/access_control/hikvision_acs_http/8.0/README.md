# Hikvision access control by HTTP

## Overview

Template for monitoring **Hikvision MinMoe face recognition terminals** (access control readers) through the **ISAPI** HTTP interface, using Zabbix HTTP agent items. No Zabbix agent or additional software is required on the device.

The template monitors:

- **Availability** — ISAPI response, HTTP status code and ICMP ping (linked *ICMP Ping* template)
- **Capacity** — registered users, faces and cards against configurable thresholds
- **Time** — clock drift, timezone and synchronization mode
- **Network** — Ethernet and wireless configuration and connection status
- **Communication** — VoIP/SIP registration and Hik-Connect cloud status
- **Security** — enabled protocols, service ports, SSH, Hik-Connect and wireless policy
- **Device information** — model, serial number, firmware and hardware version, populated into the host inventory

| | |
|---|---|
| **Author** | Caio Vernaglia |
| **Version** | 8.0-1 |
| **Zabbix** | 8.0 LTS |
| **Template group** | Templates/Access control |
| **Collection method** | HTTP agent (ISAPI, Digest authentication) |

## Requirements

### Tested devices

| Model | Firmware | Notes |
|---|---|---|
| DS-K1T343MWX | V4.48.40 build 260629 | |
| DS-K1T344MX  | V4.13.0 build 241025  | |
| DS-K1T672MX  | V3.18.0 build 240424  | SIP server discovery is not working in this version.  |

Other MinMoe models exposing the same ISAPI endpoints are expected to work but have not been tested. Reports of tested models and firmware versions are welcome.

### ISAPI endpoints used

The ISAPI user must be able to read the following endpoints:

| Endpoint | Used for |
|---|---|
| `/ISAPI/System/deviceInfo` | Device information, ISAPI status |
| `/ISAPI/System/time` | Local time, drift, timezone, sync mode |
| `/ISAPI/System/time/ntpServers` | NTP configuration |
| `/ISAPI/System/Network/interfaces` | Ethernet and wireless configuration |
| `/ISAPI/System/Network/SIP` | SIP servers (discovery) |
| `/ISAPI/System/Network/ssh` | SSH status |
| `/ISAPI/Security/adminAccesses` | Protocols and service ports |
| `/ISAPI/AccessControl/AcsWorkStatus?format=json` | Interface, VoIP and Hik-Connect status |
| `/ISAPI/AccessControl/UserInfo/Count` | Users, faces and cards count |

## Setup

1. **Create a dedicated user on the device (recommended).**
   The template uses the device's default `admin` user. For better security, create a user (e.g. `zbx_monitor`) with the lowest privilege level that can read the endpoints listed above, and set it in `{$ISAPI.USER}`.
2. **Check the HTTP(S) service.**
   Make sure HTTP or HTTPS is enabled on the device and reachable from the Zabbix server or proxy that monitors the host.
3. **Import the template** in Zabbix (**Data collection → Templates → Import**).
4. **Create the host.**
   - Add an interface with the device IP address (any interface type; the HTTP agent items use `{HOST.CONN}`).
   - Link the template **Hikvision access control by HTTP**.
   - Set **Inventory mode** to **Automatic** to populate the inventory fields.
5. **Set the host macros.**
   - Required: `{$ISAPI.PASSWORD}`.
   - If needed: `{$ISAPI.USER}`, `{$ISAPI.SCHEME}`, `{$ISAPI.PORT}`.
   - Per model: `{$CAPACITY.USERS.MAX}`, `{$CAPACITY.FACES.MAX}`, `{$CAPACITY.CARDS.MAX}` — see the device datasheet. Until these are set, capacity triggers will not fire.
   - Per installation: `{$VOIP.EXPECTED}`, `{$HIKCONNECT.ALLOWED}`, `{$SSH.ALLOWED}`, `{$WLAN.ALLOWED}`.

## Macros

| Macro | Default | Description |
|---|---|---|
| `{$ISAPI.USER}` | `admin` | ISAPI user. The device's default user; a dedicated read-only user is recommended. |
| `{$ISAPI.PASSWORD}` | _(secret)_ | ISAPI user password. |
| `{$ISAPI.SCHEME}` | `http` | ISAPI access protocol: `http` or `https`. |
| `{$ISAPI.PORT}` | `80` | ISAPI port: `80` for HTTP, `443` for HTTPS, or the device's custom port. |
| `{$CAPACITY.USERS.MAX}` | `999999` | Maximum user capacity of the model. |
| `{$CAPACITY.FACES.MAX}` | `999999` | Maximum face capacity of the model. |
| `{$CAPACITY.CARDS.MAX}` | `999999` | Maximum card capacity of the model. |
| `{$CAPACITY.THRESHOLD.INFO}` | `80` | Occupancy percentage that raises an Information alert. |
| `{$CAPACITY.THRESHOLD.WARN}` | `90` | Occupancy percentage that raises a Warning alert. |
| `{$TIME.DRIFT.MAX}` | `300` | Maximum allowed difference, in seconds, between the device clock and the Zabbix server/proxy clock. |
| `{$TIME.UTC_OFFSET}` | `-03:00` | Expected UTC offset of the device (format `±HH:MM`). |
| `{$TIME.SYNC_MODE}` | `NTP` | Expected time synchronization mode. Use `manual` when time is set by a VMS via SDK. |
| `{$VOIP.EXPECTED}` | `0` | `1` if the device operates VoIP and must be registered to the SIP server. |
| `{$HIKCONNECT.ALLOWED}` | `0` | `1` if Hik-Connect is allowed on this device. |
| `{$SSH.ALLOWED}` | `0` | `1` if SSH is allowed on this device. |
| `{$WLAN.ALLOWED}` | `0` | `1` if wireless is allowed on this device. |

Defaults follow a secure-by-default policy: Hik-Connect, SSH and wireless are considered not allowed until explicitly enabled per host.

## Items

### Raw data (master items)

Master items have no history and only feed dependent items.

| Name | Key | Interval |
|---|---|---|
| Raw: Admin accesses | `hikvision_acs.get_admin_accesses` | 1h |
| Raw: Capacity | `hikvision_acs.get_device_capacity` | 30m |
| Raw: Device info | `hikvision_acs.get_device_info` | 6h |
| Raw: Network interfaces | `hikvision_acs.get_network_interfaces` | 1m |
| Raw: NTP servers | `hikvision_acs.get_ntp_servers` | 1h |
| Raw: SIP servers | `hikvision_acs.get_sip_servers` | 2m |
| Raw: Time | `hikvision_acs.get_time` | 15m |
| Raw: Work status | `hikvision_acs.get_work_status` | 5m |

### Availability

| Name | Key | Description |
|---|---|---|
| ISAPI: Status | `hikvision_acs.isapi.status` | HTTP status code of the ISAPI service. `0` when the device does not respond. Collected every 5m. |

### Capacity

| Name | Key |
|---|---|
| Capacity: Users: Used | `hikvision_acs.capacity[users,used]` |
| Capacity: Faces: Used | `hikvision_acs.capacity[faces,used]` |
| Capacity: Cards: Used | `hikvision_acs.capacity[cards,used]` |

### Device

| Name | Key | Inventory field |
|---|---|---|
| Device: Name | `hikvision_acs.device.name` | Name |
| Device: Type | `hikvision_acs.device.type` | Type |
| Device: Manufacturer | `hikvision_acs.device.manufacturer` | Vendor |
| Device: Model | `hikvision_acs.device.model` | Model |
| Device: Serial number | `hikvision_acs.device.serial` | Serial number A |
| Device: Hardware version | `hikvision_acs.device.hardware` | Hardware |
| Device: Firmware | `hikvision_acs.device.firmware` | Software |

### Ethernet

| Name | Key | Inventory field |
|---|---|---|
| Network: Ethernet connection | `hikvision_acs.eth.connection` | |
| Ethernet: Addressing type | `hikvision_acs.eth.addressing_type` | |
| Ethernet: IP address | `hikvision_acs.eth.ip` | Host networks |
| Ethernet: Subnet mask | `hikvision_acs.eth.subnet_mask` | Host subnet mask |
| Ethernet: Gateway | `hikvision_acs.eth.gateway` | Host router |
| Ethernet: Primary DNS | `hikvision_acs.eth.dns_primary` | |
| Ethernet: Secondary DNS | `hikvision_acs.eth.dns_secondary` | |
| Ethernet: MAC address | `hikvision_acs.eth.mac` | MAC address A |

### Wireless

On devices without a wireless interface, these items return `Not present` (or the value *Not present* in `Wireless: Status`) instead of becoming unsupported, so dashboards remain consistent across models.

| Name | Key | Inventory field |
|---|---|---|
| Network: Wireless connection | `hikvision_acs.wlan.connection` | |
| Wireless: Status | `hikvision_acs.wlan.status` | |
| Wireless: SSID | `hikvision_acs.wlan.ssid` | |
| Wireless: Security mode | `hikvision_acs.wlan.security_mode` | |
| Wireless: Addressing type | `hikvision_acs.wlan.addressing_type` | |
| Wireless: IP address | `hikvision_acs.wlan.ip` | |
| Wireless: Subnet mask | `hikvision_acs.wlan.subnet_mask` | |
| Wireless: Gateway | `hikvision_acs.wlan.gateway` | |
| Wireless: Primary DNS | `hikvision_acs.wlan.dns_primary` | |
| Wireless: Secondary DNS | `hikvision_acs.wlan.dns_secondary` | |
| Wireless: MAC address | `hikvision_acs.wlan.mac` | MAC address B |

### Time

| Name | Key | Description |
|---|---|---|
| Time: Local time | `hikvision_acs.time.local` | Device local time. |
| Time: Clock drift | `hikvision_acs.time.drift` | Difference between the device clock and the Zabbix server/proxy clock, in seconds. Positive = device ahead. |
| Time: UTC offset | `hikvision_acs.time.utc_offset` | Device UTC offset. |
| Time: Sync mode | `hikvision_acs.time.sync_mode` | Time synchronization mode. |
| NTP: Server | `hikvision_acs.ntp.server` | NTP server (IP address or hostname). |
| NTP: Port | `hikvision_acs.ntp.port` | NTP server port. |
| NTP: Sync interval | `hikvision_acs.ntp.sync_interval` | NTP synchronization interval. |

### Communication

| Name | Key |
|---|---|
| VoIP: Status | `hikvision_acs.voip.status` |
| Hik-Connect: Status | `hikvision_acs.cloud.status` |

### Security

| Name | Key |
|---|---|
| Security: HTTP | `hikvision_acs.security.http.enabled` |
| Security: HTTP port | `hikvision_acs.security.http.port` |
| Security: HTTPS | `hikvision_acs.security.https.enabled` |
| Security: HTTPS port | `hikvision_acs.security.https.port` |
| Security: RTSP port | `hikvision_acs.security.rtsp.port` |
| Security: WebSocket | `hikvision_acs.security.ws.enabled` |
| Security: WebSocket port | `hikvision_acs.security.ws.port` |
| Security: WebSocketS | `hikvision_acs.security.wss.enabled` |
| Security: WebSocketS port | `hikvision_acs.security.wss.port` |
| Security: SSH | `hikvision_acs.security.ssh.enabled` |

## Discovery rules

### SIP servers discovery

Discovers the SIP servers configured on the device (`{#SIPID}`).

| Item prototype | Key |
|---|---|
| SIP server {#SIPID}: SIP address | `hikvision_acs.sip.address[{#SIPID}]` |
| SIP server {#SIPID}: SIP port | `hikvision_acs.sip.port[{#SIPID}]` |
| SIP server {#SIPID}: Display name | `hikvision_acs.sip.display_name[{#SIPID}]` |
| SIP server {#SIPID}: Extension | `hikvision_acs.sip.extension[{#SIPID}]` |
| SIP server {#SIPID}: Enabled | `hikvision_acs.sip.enabled[{#SIPID}]` |
| SIP server {#SIPID}: Expires | `hikvision_acs.sip.expires[{#SIPID}]` |
| SIP server {#SIPID}: Registration | `hikvision_acs.sip.registration[{#SIPID}]` |

## Triggers

### Availability

| Name | Severity | Description |
|---|---|---|
| ISAPI: Service is unavailable | High | No response (timeout or connection refused) or server error (5xx) in the last 2 checks. Depends on *Unavailable by ICMP ping*. |
| ISAPI: Unexpected response code | Warning | The request was rejected (4xx). Possible configuration error: credentials, user permissions or URL. Depends on *ISAPI: Service is unavailable*. |
| VoIP: Not registered | Average | VoIP is expected (`{$VOIP.EXPECTED}=1`) but not registered to the SIP server, or has been disabled. |
| Hik-Connect: Not connected | Information | Hik-Connect is enabled and allowed but not connected to the cloud. Can be closed manually. |

### Capacity

| Name | Severity | Description |
|---|---|---|
| Capacity: Users nearly full | Warning | Users above `{$CAPACITY.THRESHOLD.WARN}`% of `{$CAPACITY.USERS.MAX}`. |
| Capacity: Users usage is high | Information | Users above `{$CAPACITY.THRESHOLD.INFO}`% of `{$CAPACITY.USERS.MAX}`. |
| Capacity: Faces nearly full | Warning | Faces above `{$CAPACITY.THRESHOLD.WARN}`% of `{$CAPACITY.FACES.MAX}`. |
| Capacity: Faces usage is high | Information | Faces above `{$CAPACITY.THRESHOLD.INFO}`% of `{$CAPACITY.FACES.MAX}`. |
| Capacity: Cards nearly full | Warning | Cards above `{$CAPACITY.THRESHOLD.WARN}`% of `{$CAPACITY.CARDS.MAX}`. |
| Capacity: Cards usage is high | Information | Cards above `{$CAPACITY.THRESHOLD.INFO}`% of `{$CAPACITY.CARDS.MAX}`. |

Information triggers depend on the corresponding Warning triggers.

### Time

| Name | Severity | Description |
|---|---|---|
| Time: Clock is out of sync | Average | Device clock differs from the Zabbix server/proxy clock by more than `{$TIME.DRIFT.MAX}` seconds. |
| Time: Wrong timezone | Average | Device UTC offset differs from `{$TIME.UTC_OFFSET}`. |
| Time: Unexpected sync mode | Information | Time synchronization mode differs from `{$TIME.SYNC_MODE}`. |

### Network

| Name | Severity | Description |
|---|---|---|
| Ethernet: DHCP addressing in use | Warning | The Ethernet interface uses DHCP. A static IP address is recommended. |
| Wireless: DHCP addressing in use | Warning | Wireless is enabled and uses DHCP. |

### Security

| Name | Severity | Description |
|---|---|---|
| Hik-Connect: Enabled on a device where it is not allowed | Warning | Hik-Connect is enabled and `{$HIKCONNECT.ALLOWED}=0`. |
| Security: SSH enabled on a device where it is not allowed | Warning | SSH is enabled and `{$SSH.ALLOWED}=0`. |
| Wireless: Enabled on a device where it is not allowed | Warning | Wireless is enabled and `{$WLAN.ALLOWED}=0`. |
| Wireless: Weak security mode | Warning | Wireless is enabled with no encryption or WEP. |

### Configuration changes

These triggers must be closed manually (acknowledge to close).

| Name | Severity |
|---|---|
| Hik-Connect: Configuration has changed | Information |
| Security: SSH configuration has changed | Information |
| Security: HTTP port has changed | Information |
| Security: HTTPS port has changed | Information |
| Security: RTSP port has changed | Information |
| Security: SDK port has changed | Information |
| Security: WebSocket port has changed | Information |
| Security: WebSocketS port has changed | Information |
| Hikvision ACS: Firmware version has changed | Information |
| Hikvision ACS: Equipment has been replaced | Information |

## Graphs

| Name | Items | Notes |
|---|---|---|
| Capacity: Usage | Capacity: Users: Used, Capacity: Faces: Used, Capacity: Cards: Used | Y axis starts at 0, so changes are shown in proportion to the total. |

## Dashboards

### Hikvision acs: Overview

Single-page overview of the device, available on each host linked to the template (**Monitoring → Hosts → Dashboards**).

| Section | Widget | Content |
|---|---|---|
| Device | Host card | Host name, problem count by severity and device inventory (name, model, serial number, firmware, MAC addresses). Requires **Inventory mode = Automatic** on the host. |
| Connectivity | Item value (×8) | ISAPI, ICMP ping, ICMP response time, clock drift, Ethernet, wireless, VoIP and Hik-Connect status, colored by state. |
| Capacity | Item value (×3) | Current number of users, faces and cards. |
| Capacity graph | Graph | Users, faces and cards over the last 12 hours. Missing data is filled with the last known value. |
| ICMP response time graph | Graph | ICMP response time over the last 12 hours. |

Status colors used in the connectivity tiles:

| Tile | Green | Yellow | Red | Gray |
|---|---|---|---|---|
| ISAPI | OK (2xx) | Request rejected (4xx) | No response or server error (5xx) | |
| ICMP ping | Up | | Down | |
| ICMP response time | < 100 ms | 100–300 ms | ≥ 300 ms | |
| Clock drift | Within ±300 s | | Beyond ±300 s | |
| Ethernet | Connected | | Disconnected | |
| Wireless | Connected | | Disconnected | Not present |
| VoIP | Registered | | Not registered | |
| Hik-Connect | Connected | Disconnected | | Disabled |

The clock drift tile uses fixed thresholds matching the default `{$TIME.DRIFT.MAX}` (300 s). If the macro is changed on a host, the tile colors do not follow it.

## Known limitations

- **Ethernet connection on dashboards.** On wired-only devices, `Network: Ethernet connection` cannot report a disconnection: when the cable is down, the device is unreachable and the item keeps its last value. Use `ISAPI: Status` or ICMP ping as the online/offline indicator.
- **Port changes.** If the port used by Zabbix (`{$ISAPI.PORT}`) is changed on the device, the corresponding *port has changed* trigger cannot fire, because data collection stops first. *ISAPI: Service is unavailable* is raised instead.
- **Detection delay.** Security and port settings are collected every hour; changes may take up to 1h to be detected.
- **Clock drift.** Drift is measured against the clock of the Zabbix server or proxy performing the collection, which must be synchronized via NTP.
- **Not monitored.** Device restart/uptime, tamper and door status are not collected by this template.

## Changelog

### 8.0-1
- Initial release.
### 8.0-2
- Added support for DS-K1T344MX and DS-K1T672MX readers (see tested versions).
- Removed the SDK check, which is not supported on newer firmware versions.
- Adjusted Ethernet and wireless interface data collection for devices without wireless support.
- VoIP status now returns *Not supported* when the device does not report it.
