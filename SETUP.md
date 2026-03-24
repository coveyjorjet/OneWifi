# OneWifi Setup Guide

This guide covers building, deploying, and validating OneWifi on RDK-B platforms.

## Prerequisites

### System Requirements

- Linux build host (Ubuntu 18.04+ recommended)
- 4GB+ RAM
- 20GB+ disk space for full RDK build

### Required Packages

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y \
    build-essential \
    autoconf \
    automake \
    libtool \
    pkg-config \
    git \
    cmake \
    libev-dev \
    libnl-3-dev \
    libnl-genl-3-dev \
    libnl-route-3-dev \
    libssl-dev \
    libjansson-dev \
    libcjson-dev \
    libavro-dev \
    libprotobuf-c-dev \
    uuid-dev
```

### External Dependencies

For RDK-B builds, you need the following from your RDK sysroot:

| Dependency | Package/Repo | Purpose |
|------------|--------------|---------|
| WiFi HAL | `halinterface` | HAL interface headers |
| RDK WiFi HAL | `rdk-wifi-hal` | HAL implementation |
| hostapd | `rdk-wifi-libhostap` | WPA/802.1X authentication |
| CCSP Common | `CcspCommonLibrary` | CCSP framework |
| rbus | `rbus` | RDK Bus IPC |
| WebConfig Framework | `webconfig_framework` | WebConfig support |
| trower-base64 | `trower-base64` | Base64 encoding |

---

## Build Configuration

### Configure Options

| Option | Description | Default |
|--------|-------------|---------|
| `--enable-gtestapp` | Build Google Test suite | off |
| `--enable-libwebconfig` | Build WebConfig library | off |
| `--enable-easymesh` | Enable EasyMesh support | off |
| `--enable-sm-app` | Enable Statistics Manager | off |
| `--enable-em-app` | Enable EasyMesh app | off |
| `--enable-easyconnect` | Enable DPP/EasyConnect | off |
| `--enable-notify` | Enable systemd notify | off |
| `--enable-journalctl` | Enable journalctl logging | off |
| `--with-ccsp-arch=ARG` | CPU architecture (arm/atom/pc/mips) | none |
| `--with-ccsp-platform=ARG` | CCSP platform (intel_usg/pc/bcm) | none |

### Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `PKG_CONFIG_SYSROOT_DIR` | RDK sysroot path | `/opt/rdk/sysroot` |
| `DEVICE_EXTENDER` | Build for extender device | `true` |
| `HAL_IPC` | Use HAL IPC mode | `true` |
| `ONEWIFI_DML_SUPPORT_MAKEFILE` | Enable DML support | `true` |
| `ONEWIFI_DBUS_SUPPORT` | Use D-Bus instead of rbus | `true` |

### Conditional Compile Flags

| Flag | Purpose |
|------|---------|
| `-DFEATURE_ONE_WIFI` | OneWifi feature enabled |
| `-DWIFI_HAL_VERSION_3` | WiFi HAL version 3 |
| `-DFEATURE_SUPPORT_PASSPOINT` | Passpoint/HS2.0 support |
| `-DFEATURE_SUPPORT_WEBCONFIG` | WebConfig support |
| `-DENABLE_FEATURE_MESHWIFI` | Mesh WiFi feature |
| `-D_ANSC_LITTLE_ENDIAN_` | Little-endian arch |

---

## Build Instructions

### Standard RDK-B Build

```bash
# Clone repository
git clone https://github.com/rdkcentral/OneWifi.git
cd OneWifi

# Generate autotools files
./autogen.sh

# Configure for ARM RDK-B
export PKG_CONFIG_SYSROOT_DIR=/path/to/rdk-sysroot
./configure \
    --prefix=/usr \
    --enable-libwebconfig \
    --enable-easymesh \
    --enable-sm-app \
    --enable-em-app \
    --with-ccsp-arch=arm \
    --with-ccsp-platform=intel_usg

# Build
make -j$(nproc)

# Install to staging directory
make DESTDIR=/tmp/onewifi-staging install
```

### Build with Test Support

```bash
./configure \
    --enable-gtestapp \
    --enable-libwebconfig \
    --with-ccsp-arch=arm

make

# Run tests (on build host or target)
./OneWifi_gtest.bin
```

### Linux Platform Build (Raspberry Pi)

For standalone Linux builds without full RDK stack:

```bash
# First time: download dependencies
make -f build/linux/rpi/makefile setup

# Build
make -f build/linux/rpi/makefile all

# Binaries in install/bin/
ls install/bin/
```

### Linux Platform Build (Banana Pi)

```bash
# Setup (downloads hostapd 2.11, rdk-wifi-hal, etc.)
make -f build/linux/bpi/makefile setup

# Build
make -f build/linux/bpi/makefile all
```

---

## Deployment

### RDK-B Deployment

**Copy binaries to device**:
```bash
# Main binary
scp OneWifi root@<device>:/usr/bin/

# Libraries (if needed)
scp libwifi*.so* root@<device>:/usr/lib/

# Scripts
scp scripts/*.sh root@<device>:/usr/ccsp/wifi/
```

**Service files**:
```bash
# systemd service
scp scripts/onewifi.service root@<device>:/lib/systemd/system/
ssh root@<device> "systemctl daemon-reload"
```

### Service Management

```bash
# Start service
systemctl start onewifi.service

# Stop service
systemctl stop onewifi.service

# Restart service
systemctl restart onewifi.service

# Check status
systemctl status onewifi.service

# View logs
journalctl -u onewifi.service -f
```

### Manual Execution (Debug)

```bash
# Run in foreground (console mode)
/usr/bin/OneWifi -c

# Run in background (daemon mode)
/usr/bin/OneWifi &
```

---

## Configuration

### Database Initialization

On first boot, OVSDB schema is initialized:

```bash
# Database location
/opt/secure/wifi/rdkb-wifi.db

# Schema files
/usr/ccsp/wifi/*.ovsschema
```

### Bootstrap Configuration

Initial configuration can be provided via:
```bash
/opt/secure/bootstrap.json
```

### Interface Mapping

Configure WiFi interfaces in:
```bash
/nvram/InterfaceMap.json
```

---

## Runtime Validation

### Service Health Check

```bash
# Check process
pidof OneWifi

# Check systemd status
systemctl status onewifi.service

# Check for errors in logs
cat /rdklogs/logs/WifiLog.txt.0 | grep -i "error\|failed"
```

### TR-181 Queries

```bash
# Get radio count
dmcli eRT getv Device.WiFi.RadioNumberOfEntries

# Get radio status
dmcli eRT getv Device.WiFi.Radio.1.Enable
dmcli eRT getv Device.WiFi.Radio.1.Channel
dmcli eRT getv Device.WiFi.Radio.1.OperatingChannelBandwidth

# Get SSID info
dmcli eRT getv Device.WiFi.SSID.1.SSID
dmcli eRT getv Device.WiFi.SSID.1.Enable

# Get security mode
dmcli eRT getv Device.WiFi.AccessPoint.1.Security.ModeEnabled

# Set parameters
dmcli eRT setv Device.WiFi.Radio.1.Channel uint 6
dmcli eRT setv Device.WiFi.SSID.1.SSID string "MyNetwork"
```

### rbus Queries

```bash
# Get value
rbuscli get Device.WiFi.Radio.1.Enable

# Set value
rbuscli set Device.WiFi.Radio.1.Channel uint32 6

# Subscribe to events
rbuscli sub Device.WiFi.AssociatedDevice.1.SignalStrength
```

### OVSDB Queries

```bash
# Dump all radio configs
ovsdb-client dump Wifi_Radio_Config

# Dump VAP configs
ovsdb-client dump Wifi_VAP_Config

# Query specific table
ovsdb-client transact '["Open_vSwitch",{
    "op": "select",
    "table": "Wifi_Radio_Config",
    "where": []
}]'

# Monitor changes
ovsdb-client monitor Wifi_Radio_Config
```

### WiFi Status Commands

```bash
# Check interfaces
iw dev

# Check associated clients
iw dev wlan0 station dump

# Check channel
iw dev wlan0 info

# Scan for networks
iw dev wlan0 scan
```

---

## Debugging

### Enable Debug Logging

```bash
# Enable controller debug
touch /nvram/wifiCtrlDbg

# Enable WebConfig debug
touch /nvram/wifiWebConfigDbg

# Enable database debug
touch /nvram/wifiDbDbg

# Enable all modules
for m in wifiCtrlDbg wifiMgrDbg wifiWebConfigDbg wifiDbDbg wifiMonDbg wifiAppsDbg; do
    touch /nvram/$m
done
```

### Log Locations

| Log | Path |
|-----|------|
| WiFi Log | `/rdklogs/logs/WifiLog.txt.0` |
| Arm Console | `/rdklogs/logs/ArmConsolelog.txt.0` |
| System Log | `/var/log/messages` |
| Journal | `journalctl -u onewifi.service` |

### Log Analysis

```bash
# Follow live logs
tail -f /rdklogs/logs/WifiLog.txt.0

# Filter by module
grep "wifiCtrl" /rdklogs/logs/WifiLog.txt.0

# Find errors
grep -i "error\|failed\|exception" /rdklogs/logs/WifiLog.txt.0

# Trace event flow
grep -i "push_event\|handle.*event" /rdklogs/logs/WifiLog.txt.0
```

### Core Dump Analysis

```bash
# Enable core dumps
sysctl -w kernel.core_pattern="/tmp/cores/core.%e.%p.%t"
mkdir -p /tmp/cores
ulimit -c unlimited

# Analyze core
gdb /usr/bin/OneWifi /tmp/cores/core.OneWifi.*
(gdb) bt full
(gdb) info threads
(gdb) thread apply all bt
```

### GDB Attach

```bash
# Find PID
PID=$(pidof OneWifi)

# Attach
gdb -p $PID

# Set breakpoints
(gdb) break wifi_ctrl_queue_handlers.c:4317
(gdb) break webconfig_decode
(gdb) continue
```

---

## Troubleshooting

### Service Won't Start

```bash
# Check for missing libraries
ldd /usr/bin/OneWifi | grep "not found"

# Check permissions
ls -la /usr/bin/OneWifi
ls -la /opt/secure/wifi/

# Check database
ls -la /opt/secure/wifi/rdkb-wifi.db

# Run manually with debug
/usr/bin/OneWifi -c 2>&1 | tee /tmp/onewifi.log
```

### Configuration Not Applying

```bash
# Check webconfig state
grep "webconfig_state\|pending" /rdklogs/logs/WifiLog.txt.0

# Force config reload
rbuscli set Device.WiFi.WebConfig.Data.Subdoc.North string '{...}'

# Check OVSDB sync
ovsdb-client dump Wifi_Radio_Config | grep -i channel
```

### Client Connection Issues

```bash
# Check AP status
iw dev wlan0 info

# Check hostapd
ps aux | grep hostapd
cat /var/run/hostapd/wlan0.conf

# Check associated clients
iw dev wlan0 station dump

# Check MAC filter
ovsdb-client dump Wifi_MacFilter_Config
```

### HAL Issues

```bash
# Check HAL library
ldd /usr/bin/OneWifi | grep hal

# Check HAL callbacks
grep "hal_callback\|wifi_hal_" /rdklogs/logs/WifiLog.txt.0

# Verify HAL init
grep "wifi_hal_init\|getHalCapability" /rdklogs/logs/WifiLog.txt.0
```

---

## Example Workflows

### Change WiFi Channel

```bash
# Via TR-181
dmcli eRT setv Device.WiFi.Radio.1.Channel uint 36

# Via rbus
rbuscli set Device.WiFi.Radio.1.Channel uint32 36

# Verify
dmcli eRT getv Device.WiFi.Radio.1.Channel
iw dev wlan0 info | grep channel
```

### Update SSID

```bash
# Set new SSID
dmcli eRT setv Device.WiFi.SSID.1.SSID string "NewNetworkName"

# Verify
dmcli eRT getv Device.WiFi.SSID.1.SSID
iw dev wlan0 info | grep ssid
```

### Enable/Disable Radio

```bash
# Disable radio
dmcli eRT setv Device.WiFi.Radio.1.Enable bool false

# Enable radio
dmcli eRT setv Device.WiFi.Radio.1.Enable bool true

# Verify
iw dev
```

### Factory Reset WiFi

```bash
# Via TR-181 (if supported)
dmcli eRT setv Device.WiFi.Reset bool true

# Manual reset
rm -f /opt/secure/wifi/rdkb-wifi.db
systemctl restart onewifi.service
```

---

## Performance Tuning

### Scheduler Tuning

Edit poll timeout in `wifi_ctrl.c`:
```c
#define CTRL_QUEUE_POLL_TIMEOUT_MS  200  // Default 200ms
```

### Event Queue Size

Default queue sizes in `wifi_ctrl.c`:
```c
#define MAX_QUEUE_SIZE  1024  // Max events in queue
```

### Memory Optimization

For memory-constrained devices:
```bash
# Disable unused apps at compile time
./configure --disable-blaster --disable-harvester ...
```

---

## Security Considerations

- Database files in `/opt/secure/wifi/` should have restricted permissions (0600)
- Passphrase storage is encrypted via secure storage APIs
- Debug logging may expose sensitive information - disable in production
- Core dumps may contain sensitive data - secure `/tmp/cores/`
