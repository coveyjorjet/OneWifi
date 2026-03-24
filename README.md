# OneWifi

Unified Wi-Fi management service for RDK-B devices. Manages Wi-Fi parameters, statistics, telemetry, client steering, and optimization for both Gateways and Extenders.

## Overview

OneWifi is a CCSP (Common Component Software Platform) component that implements the TR-181 Device.WiFi data model. It provides:

- **Radio/VAP Management**: Configure radios, SSIDs, security, and access points
- **WebConfig Integration**: Cloud-based configuration via subdocuments
- **Multi-AP/EasyMesh**: IEEE 1905.1 mesh networking support
- **Client Steering**: Band steering and load balancing
- **Passpoint/Hotspot 2.0**: Carrier Wi-Fi offload support
- **Statistics & Telemetry**: Real-time metrics collection and reporting

## Position in RDK-B Stack

```
┌─────────────────────────────────────────────────────┐
│              Cloud / WebConfig Server               │
└─────────────────────┬───────────────────────────────┘
                      │ WebConfig Subdocs
┌─────────────────────▼───────────────────────────────┐
│                    OneWifi                          │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌───────────┐ │
│  │ TR-181  │ │WebConfig│ │  rbus   │ │   OVSDB   │ │
│  │  DML    │ │ Decoder │ │  IPC    │ │   Store   │ │
│  └─────────┘ └─────────┘ └─────────┘ └───────────┘ │
└─────────────────────┬───────────────────────────────┘
                      │ WiFi HAL API
┌─────────────────────▼───────────────────────────────┐
│              WiFi HAL (rdk-wifi-hal)                │
│         + hostapd/wpa_supplicant integration        │
└─────────────────────┬───────────────────────────────┘
                      │ nl80211/cfg80211
┌─────────────────────▼───────────────────────────────┐
│              Kernel WiFi Drivers                    │
└─────────────────────────────────────────────────────┘
```

## Quick Start

### Build (RDK-B)

```bash
./autogen.sh
./configure \
    --enable-gtestapp \
    --enable-libwebconfig \
    --enable-easymesh \
    --with-ccsp-arch=arm
make
```

### Deploy

```bash
# Copy binary to target
scp OneWifi root@<device>:/usr/bin/

# Start service
systemctl start onewifi.service
systemctl status onewifi.service
```

### Validate

```bash
# Check TR-181 parameters
dmcli eRT getv Device.WiFi.RadioNumberOfEntries
dmcli eRT getv Device.WiFi.Radio.1.Channel

# View logs
cat /rdklogs/logs/WifiLog.txt.0 | tail -50
```

## Key Features

| Feature | Description | Configure Flag |
|---------|-------------|----------------|
| WebConfig | Cloud configuration subdocs | `--enable-libwebconfig` |
| EasyMesh | Multi-AP mesh networking | `--enable-easymesh` |
| Statistics Manager | Telemetry collection | `--enable-sm-app` |
| EasyConnect (DPP) | Device provisioning | `--enable-easyconnect` |
| Passpoint | Hotspot 2.0 support | Built-in |
| Google Test | Unit testing | `--enable-gtestapp` |

## Documentation

- [ARCHITECTURE.md](ARCHITECTURE.md) - System design, IPC, data flow
- [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) - Code navigation, extending OneWifi
- [SETUP.md](SETUP.md) - Build environment, deployment, debugging
- [AGENTS.md](AGENTS.md) - AI agent coding guidelines
- [CODE_STYLE.md](CODE_STYLE.md) - C code style conventions

## Project Structure

```
source/
├── core/           # Main control logic (wifi_ctrl, wifi_mgr)
├── apps/           # Feature apps (analytics, blaster, CSI, EM, SM)
├── db/             # OVSDB database layer
├── dml/            # TR-181 Data Model Layer
├── platform/       # Platform-specific code (rdkb, linux, dbus)
├── webconfig/      # WebConfig subdoc handlers
├── stats/          # Statistics collection
└── utils/          # Utilities (scheduler, validator)
lib/                # Internal libraries (ovsdb, ds, schema)
include/            # Public headers
build/              # Build configurations
```

## License

Apache License 2.0 - See [LICENSE](LICENSE) for details.

Copyright 2018 RDK Management
