# OneWifi Developer Guide

This guide helps developers navigate the codebase, understand key modules, and extend OneWifi functionality.

## Codebase Navigation

### Entry Points

| Purpose | File | Function | Line |
|---------|------|----------|------|
| Main entry | `source/core/wifi_mgr.c` | `main()` | 373 |
| Event loop | `source/core/wifi_ctrl.c` | `ctrl_queue_loop()` | 303 |
| Event handlers | `source/core/wifi_ctrl_queue_handlers.c` | `handle_*_event()` | Various |
| WebConfig decode | `source/webconfig/wifi_webconfig.c` | `webconfig_decode()` | 67 |
| Database ops | `source/db/wifi_db_apis.c` | `wifidb_*()` | Various |
| rbus handlers | `source/core/wifi_ctrl_rbus_handlers.c` | Data element array | 3994 |

### Key Directories

```
source/
├── core/                   # Control plane
│   ├── wifi_mgr.c         # Entry point, initialization
│   ├── wifi_ctrl.c        # Event loop, scheduler
│   ├── wifi_ctrl_queue_handlers.c  # Event dispatch (4500 lines)
│   ├── wifi_ctrl_rbus_handlers.c   # rbus data elements
│   ├── wifi_ctrl_webconfig.c       # WebConfig apply logic
│   └── services/          # VAP services (private, public, mesh)
│
├── apps/                   # Feature applications
│   ├── wifi_apps_mgr.c    # App registration and event forwarding
│   ├── analytics/         # Telemetry collection
│   ├── blaster/           # WiFi performance testing
│   ├── csi/               # Channel State Information
│   ├── em/                # EasyMesh
│   ├── sm/                # Statistics Manager
│   ├── harvester/         # Data harvesting
│   └── ...
│
├── webconfig/              # Configuration handlers
│   ├── wifi_webconfig.c   # Core subdoc framework
│   ├── wifi_decoder.c     # JSON decoding
│   ├── wifi_encoder.c     # JSON encoding
│   ├── wifi_webconfig_private.c   # Private SSID subdoc
│   ├── wifi_webconfig_radio.c     # Radio subdoc
│   └── ...                # 39 subdoc handlers
│
├── db/                     # Database layer
│   ├── wifi_db.h          # Interface definitions
│   ├── wifi_db.c          # Default implementation
│   └── wifi_db_apis.c     # OVSDB implementation
│
├── dml/                    # TR-181 Data Model Layer
│   ├── tr_181/ml/         # Middle layer (cosa_wifi_dml.c)
│   └── tr_181/sbapi/      # Southbound API
│
├── platform/               # Platform abstraction
│   ├── rdkb/bus.c         # rbus implementation
│   ├── linux/bus.c        # HE-Bus (Linux)
│   ├── dbus/bus.c         # D-Bus (legacy)
│   └── common/            # Shared platform code
│
├── utils/                  # Utilities
│   ├── scheduler.c        # Timer scheduler
│   ├── wifi_util.c        # Logging, helpers
│   └── wifi_validator.c   # Config validation
│
└── stats/                  # Statistics collection
    └── wifi_monitor.c     # WiFi monitoring
```

### Key Header Files

| Header | Purpose |
|--------|---------|
| `include/wifi_base.h` | Core types, RFC flags, global structures |
| `include/wifi_webconfig.h` | WebConfig types, subdoc definitions |
| `source/core/wifi_ctrl.h` | Controller state, event types, state machine |
| `source/core/wifi_mgr.h` | Manager structure |
| `source/utils/wifi_util.h` | Logging macros, utility functions |

---

## Module Deep-Dives

### Event System

Events flow through a central queue processed by `ctrl_queue_loop()`.

**Creating an Event** (`source/core/wifi_events.c`):
```c
wifi_event_t *event = create_wifi_event(
    sizeof(my_data_t),           // Data size
    wifi_event_type_hal_ind,     // Event type
    wifi_event_hal_assoc_device  // Subtype
);
memcpy(event->u.core_data.msg, &my_data, sizeof(my_data_t));
push_event_to_ctrl_queue(event);
```

**Event Types** (`source/core/wifi_ctrl.h`):
```c
typedef enum {
    wifi_event_type_webconfig,    // Configuration events
    wifi_event_type_hal_ind,      // HAL indications
    wifi_event_type_command,      // Command events
    wifi_event_type_wifiapi,      // WiFi API events
    wifi_event_type_monitor,      // Monitor events
    wifi_event_type_analytic,     // Analytics events
    wifi_event_type_exec,         // Execution events
} wifi_event_type_t;
```

### Scheduler System

**Adding a Timer Task** (`source/utils/scheduler.c`):
```c
scheduler_add_timer_task(
    ctrl->scheduler,
    FALSE,                    // Not high priority
    &task_id,                 // Output: task ID
    my_callback,              // Callback function
    callback_arg,             // Callback argument
    5000,                     // Interval: 5 seconds
    0,                        // Repeat forever (0 = infinite)
    FALSE                     // Don't schedule immediately
);
```

**Callback Signature**:
```c
int my_callback(void *arg) {
    // Do work
    return TIMER_TASK_COMPLETE;  // or TIMER_TASK_CONTINUE
}
```

### WebConfig Subdocs

Each subdoc has a handler implementing the subdoc interface.

**File Pattern**: `source/webconfig/wifi_webconfig_<name>.c`

**Handler Functions**:
| Function | Purpose |
|----------|---------|
| `init_<name>_subdoc()` | Initialize subdoc handler |
| `encode_<name>_subdoc()` | Struct → JSON |
| `decode_<name>_subdoc()` | JSON → Struct |
| `translate_to_<name>_subdoc()` | External → Internal format |
| `translate_from_<name>_subdoc()` | Internal → External format |

---

## Extending OneWifi

### Adding a New WebConfig Subdoc

**Step 1**: Create subdoc handler file

```c
// source/webconfig/wifi_webconfig_myfeature.c

#include "wifi_webconfig.h"

webconfig_error_t init_myfeature_subdoc(webconfig_subdoc_t *doc)
{
    doc->type = webconfig_subdoc_type_myfeature;
    doc->name = "myfeature";
    doc->version = 1;
    return webconfig_error_none;
}

webconfig_error_t encode_myfeature_subdoc(webconfig_t *config, 
                                          webconfig_subdoc_data_t *data)
{
    cJSON *root = cJSON_CreateObject();
    // Add fields to JSON
    cJSON_AddStringToObject(root, "SubDocName", "myfeature");
    cJSON_AddNumberToObject(root, "Version", 1);
    // ... add your data fields
    
    data->u.encoded.raw = cJSON_PrintUnformatted(root);
    cJSON_Delete(root);
    return webconfig_error_none;
}

webconfig_error_t decode_myfeature_subdoc(webconfig_t *config,
                                          webconfig_subdoc_data_t *data)
{
    cJSON *root = cJSON_Parse(data->u.encoded.raw);
    if (!root) {
        return webconfig_error_decode;
    }
    // Parse JSON fields into data->u.decoded.*
    cJSON_Delete(root);
    return webconfig_error_none;
}
```

**Step 2**: Add type to enum (`include/wifi_webconfig.h`):
```c
typedef enum {
    // ... existing types ...
    webconfig_subdoc_type_myfeature,
    webconfig_subdoc_type_max
} webconfig_subdoc_type_t;
```

**Step 3**: Register in `wifi_webconfig.c`:
```c
// In webconfig_init(), around line 300
config->subdocs[webconfig_subdoc_type_myfeature].init_subdoc = init_myfeature_subdoc;
config->subdocs[webconfig_subdoc_type_myfeature].encode_subdoc = encode_myfeature_subdoc;
config->subdocs[webconfig_subdoc_type_myfeature].decode_subdoc = decode_myfeature_subdoc;
```

**Step 4**: Add apply function (`wifi_ctrl_webconfig.c`):
```c
int webconfig_hal_myfeature_apply(webconfig_subdoc_data_t *data)
{
    // Apply configuration to HAL
    // Update OVSDB if needed
    return 0;
}
```

**Step 5**: Update Makefile.am:
```makefile
libwifi_webconfig_la_SOURCES += wifi_webconfig_myfeature.c
```

---

### Adding a New App Module

**Step 1**: Create app directory and files

```
source/apps/myapp/
├── wifi_myapp.c
├── wifi_myapp.h
└── Makefile.am
```

**Step 2**: Implement app interface (`wifi_myapp.c`):

```c
#include "wifi_apps_mgr.h"
#include "wifi_myapp.h"

static int myapp_init(wifi_app_t *app, wifi_mgr_t *mgr)
{
    wifi_util_info_print(WIFI_APPS, "%s:%d: MyApp initialized\n", 
                         __func__, __LINE__);
    return 0;
}

static int myapp_deinit(wifi_app_t *app)
{
    return 0;
}

static int myapp_event(wifi_app_t *app, wifi_event_t *event)
{
    switch (event->event_type) {
    case wifi_event_type_hal_ind:
        // Handle HAL indication
        break;
    case wifi_event_type_webconfig:
        // Handle config change
        break;
    default:
        break;
    }
    return 0;
}

wifi_app_t wifi_app_myapp = {
    .app_id = wifi_app_inst_myapp,
    .app_name = "MyApp",
    .init_fn = myapp_init,
    .deinit_fn = myapp_deinit,
    .event_fn = myapp_event,
};
```

**Step 3**: Add to apps manager (`wifi_apps_mgr.c`):

```c
// In apps_mgr_init()
#ifdef ONEWIFI_MYAPP_SUPPORT
extern wifi_app_t wifi_app_myapp;
apps_mgr_register_app(mgr, &wifi_app_myapp);
#endif
```

**Step 4**: Add compile flag (`configure.ac`):
```
AC_ARG_ENABLE([myapp],
    AS_HELP_STRING([--enable-myapp],[enable myapp]),
    [ONEWIFI_MYAPP_SUPPORT=true],
    [ONEWIFI_MYAPP_SUPPORT=false])
AM_CONDITIONAL([ONEWIFI_MYAPP_SUPPORT], [test x$ONEWIFI_MYAPP_SUPPORT = xtrue])
```

---

### Adding a New rbus Data Element

**Step 1**: Define handler functions (`wifi_ctrl_rbus_handlers.c`):

```c
bus_error_t myfeature_get_handler(char *event_name, raw_data_t *p_data, 
                                   bus_user_data_t *user_data)
{
    // Get current value
    char *value = get_my_value();
    
    p_data->data_type = bus_data_type_string;
    p_data->raw_data.bytes = strdup(value);
    p_data->raw_data_len = strlen(value) + 1;
    
    return bus_error_success;
}

bus_error_t myfeature_set_handler(char *event_name, raw_data_t *p_data,
                                   bus_user_data_t *user_data)
{
    // Validate and apply new value
    char *new_value = (char *)p_data->raw_data.bytes;
    
    if (apply_my_value(new_value) != 0) {
        return bus_error_invalid_input;
    }
    
    return bus_error_success;
}
```

**Step 2**: Add to data element array (around line 3994):

```c
static wifi_bus_desc_t bus_data_elements[] = {
    // ... existing elements ...
    {
        "Device.WiFi.MyFeature.Value",
        HOSTIF_None,
        { myfeature_get_handler, myfeature_set_handler, NULL, NULL, NULL, NULL },
        slow_speed,
        ZERO_TABLE,
    },
};
```

**Step 3**: Test with rbuscli:
```bash
rbuscli get Device.WiFi.MyFeature.Value
rbuscli set Device.WiFi.MyFeature.Value string "newvalue"
```

---

## Debugging Techniques

### Enabling Debug Logging

Debug logging is controlled by marker files in `/nvram/`:

| Module | Enable File | Log Identifier |
|--------|-------------|----------------|
| Controller | `/nvram/wifiCtrlDbg` | `wifiCtrl` |
| Manager | `/nvram/wifiMgrDbg` | `wifiMgr` |
| WebConfig | `/nvram/wifiWebConfigDbg` | `wifiWebConfig` |
| Database | `/nvram/wifiDbDbg` | `wifiDb` |
| Monitor | `/nvram/wifiMonDbg` | `wifiMon` |
| Apps | `/nvram/wifiAppsDbg` | `wifiApps` |
| Passpoint | `/nvram/wifiPasspointDbg` | `wifiPasspoint` |

**Enable debug for a module**:
```bash
touch /nvram/wifiCtrlDbg
# Restart or wait for log rotation
```

**View logs**:
```bash
# RDK-B
cat /rdklogs/logs/WifiLog.txt.0 | grep -i "wifiCtrl"

# Filter errors
cat /rdklogs/logs/WifiLog.txt.0 | grep -i "error\|failed"

# Follow live
tail -f /rdklogs/logs/WifiLog.txt.0
```

### Common Debug Patterns

**Tracing Event Flow**:
```bash
# Watch for WebConfig events
grep -i "handle_webconfig_event\|webconfig_decode" /rdklogs/logs/WifiLog.txt.0

# Watch for HAL events
grep -i "hal_ind\|assoc_device\|disassoc" /rdklogs/logs/WifiLog.txt.0
```

**Checking Configuration**:
```bash
# TR-181 queries
dmcli eRT getv Device.WiFi.Radio.1.
dmcli eRT getv Device.WiFi.SSID.1.

# OVSDB queries
ovsdb-client dump Wifi_Radio_Config
ovsdb-client dump Wifi_VAP_Config
```

**Forcing Configuration Reload**:
```bash
# Via rbus
rbuscli set Device.WiFi.WebConfig.Data.Subdoc.North string '{"SubDocName":"radio",...}'

# Restart service
systemctl restart onewifi.service
```

### GDB Debugging

```bash
# Attach to running process
gdb -p $(pidof OneWifi)

# Set breakpoints
(gdb) break ctrl_queue_loop
(gdb) break handle_webconfig_event
(gdb) break wifi_hal_createVAP

# Continue
(gdb) continue
```

### Core Dump Analysis

```bash
# Enable core dumps (if not already)
sysctl -w kernel.core_pattern="/tmp/cores/core.%e.%p.%t"
ulimit -c unlimited

# Analyze core
gdb /usr/bin/OneWifi /tmp/cores/core.OneWifi.*
(gdb) bt full
```

---

## Testing

### Building with GTest

```bash
./configure --enable-gtestapp
make
```

### Running Tests

```bash
# All tests
./OneWifi_gtest.bin

# Specific test
./OneWifi_gtest.bin --gtest_filter=WebConfigTest.*
./OneWifi_gtest.bin --gtest_filter=RadioConfigTest.ChannelChange

# Verbose output
./OneWifi_gtest.bin --gtest_filter=* --gtest_print_time=1
```

### Test Output

XML reports generated at: `/tmp/Gtest_Report/OneWifi_gtest_report.xml`

### Adding Unit Tests

Create test file in `source/test/`:

```cpp
// source/test/myfeature_test.cpp

#include <gtest/gtest.h>
extern "C" {
#include "wifi_webconfig.h"
}

class MyFeatureTest : public ::testing::Test {
protected:
    void SetUp() override {
        // Setup code
    }
    
    void TearDown() override {
        // Cleanup
    }
};

TEST_F(MyFeatureTest, BasicFunctionality) {
    // Test code
    EXPECT_EQ(my_function(), expected_value);
}

TEST_F(MyFeatureTest, ErrorHandling) {
    EXPECT_EQ(my_function(NULL), -1);
}
```

---

## Common Pitfalls

### Thread Safety

- Always use `associated_devices_lock` when accessing `associated_devices_map`
- Event queue operations are thread-safe (use `push_event_to_ctrl_queue()`)
- Scheduler operations should be done from main thread context

### Memory Management

- WebConfig data must be freed with `webconfig_data_free()`
- Event data is automatically freed after handler returns
- OVSDB results must be freed by caller

### HAL Callbacks

- HAL callbacks run in HAL thread context
- Must push events to queue, not process directly
- Avoid blocking operations in callbacks

### Configuration Persistence

- Always update OVSDB after applying changes to HAL
- Use `wifidb_update_*` functions for consistency
- Check return values and handle failures

---

## Code Style Reference

See [CODE_STYLE.md](CODE_STYLE.md) for detailed conventions.

**Quick Reference**:
- Indentation: 4 spaces
- Line length: 100 characters
- Braces: K&R for control, new line for functions
- Naming: `snake_case` for functions/variables, `UPPER_CASE` for macros
- Always use braces for control statements
- Use `wifi_util_*_print()` for logging
