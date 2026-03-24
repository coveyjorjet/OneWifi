# OneWifi Architecture

This document describes the internal architecture, IPC mechanisms, data flow, and key components of OneWifi.

## System Architecture

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL INTERFACES                               │
├──────────────────┬──────────────────┬─────────────────┬───────────────────────┤
│    TR-181/CCSP   │    WebConfig     │      rbus       │        OVSDB          │
│  (dmcli, SNMP)   │  (Cloud Config)  │   (RDK IPC)     │   (Persistent DB)     │
└────────┬─────────┴────────┬─────────┴────────┬────────┴──────────┬────────────┘
         │                  │                  │                   │
┌────────▼──────────────────▼──────────────────▼───────────────────▼────────────┐
│                              OneWifi Core                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                         wifi_ctrl (Event Loop)                          │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │  │
│  │  │ Event Queue  │  │  Scheduler   │  │  Webconfig   │  │    State    │  │  │
│  │  │   Handler    │  │   (Timers)   │  │   Handler    │  │   Machine   │  │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────────────┘  │  │
│  └─────────┼─────────────────┼─────────────────┼───────────────────────────┘  │
│            │                 │                 │                              │
│  ┌─────────▼─────────────────▼─────────────────▼───────────────────────────┐  │
│  │                        Apps Manager                                      │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────┐ │  │
│  │  │Analytics│ │ Blaster │ │   CSI   │ │Harvester│ │   SM    │ │   EM   │ │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘ └────────┘ │  │
│  └──────────────────────────────┬──────────────────────────────────────────┘  │
│                                 │                                             │
│  ┌──────────────────────────────▼──────────────────────────────────────────┐  │
│  │                    Platform Abstraction Layer                            │  │
│  │     ┌──────────────┐    ┌──────────────┐    ┌──────────────┐            │  │
│  │     │  RDKB/rbus   │    │ Linux/HE-Bus │    │    D-Bus     │            │  │
│  │     │   bus.c      │    │    bus.c     │    │    bus.c     │            │  │
│  │     └──────────────┘    └──────────────┘    └──────────────┘            │  │
│  └──────────────────────────────┬──────────────────────────────────────────┘  │
└─────────────────────────────────┼──────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────▼──────────────────────────────────────────────┐
│                              WiFi HAL Layer                                    │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │   Callbacks: assoc/disassoc, scan results, channel change, WPS, frames   │ │
│  │   Operations: createVAP, setRadioParams, startScan, connect/disconnect   │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
│                                    │                                          │
│  ┌─────────────────────────────────▼────────────────────────────────────────┐ │
│  │                        hostapd Integration                                │ │
│  │            (EAP, WPA, 802.1X, RADIUS, Passpoint)                         │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────────┘
```

## Core Modules

### 1. wifi_mgr (Manager)

**File**: `source/core/wifi_mgr.c`

Entry point and initialization coordinator.

| Function | Line | Purpose |
|----------|------|---------|
| `main()` | 373 | Entry point, daemonization, init sequence |
| `init_wifimgr()` | 248 | HAL init, radio config, control module init |
| `start_wifimgr()` | 340 | Start apps manager and control loop |

**Initialization Sequence**:
```
main() → platform_init() → daemonize_fn() → init_wifimgr()
    → wifi_hal_pre_init()
    → wifi_hal_init()
    → wifi_hal_getHalCapability()
    → init_global_radio_config()
    → init_wifi_ctrl()
    → wifidb_init()
```

### 2. wifi_ctrl (Controller)

**File**: `source/core/wifi_ctrl.c`

Event loop and central coordination.

| Function | Line | Purpose |
|----------|------|---------|
| `init_wifi_ctrl()` | 142 | Initialize scheduler, queues, webconfig, HAL callbacks |
| `ctrl_queue_loop()` | 303 | Main event loop (pthread-based) |
| `push_event_to_ctrl_queue()` | 270 | Push events to processing queue |

**Event Loop Mechanism**:
```c
// NOT libev - custom pthread-based loop
while (running) {
    pthread_cond_timedwait(&ctrl->cond, &ctrl->lock, &timeout);
    
    while (!queue_empty(ctrl->queue)) {
        event = queue_pop(ctrl->queue);
        dispatch_event(event);  // → wifi_ctrl_queue_handlers.c
    }
    
    scheduler_execute();  // Run timer tasks
}
```

**Poll Timeout**: 200ms default (configurable)

### 3. wifi_ctrl_queue_handlers (Event Handlers)

**File**: `source/core/wifi_ctrl_queue_handlers.c` (~4500 lines)

Handles all event types dispatched from the event loop.

| Event Type | Handler | Line |
|------------|---------|------|
| `wifi_event_type_webconfig` | `handle_webconfig_event()` | 4317 |
| `wifi_event_type_hal_ind` | `handle_hal_indication()` | 4168 |
| `wifi_event_type_command` | `handle_command_event()` | 4200 |
| `wifi_event_type_wifiapi` | `handle_wifiapi_event()` | 4476 |
| `wifi_event_type_monitor` | `handle_monitor_event()` | 4491 |

**HAL Indication Sub-handlers**:
| Subtype | Handler | Purpose |
|---------|---------|---------|
| `wifi_event_hal_assoc_device` | `process_assoc_device_event()` | Client connected |
| `wifi_event_hal_disassoc_device` | `process_disassoc_device_event()` | Client disconnected |
| `wifi_event_hal_channel_change` | `process_channel_change_event()` | Channel changed |
| `wifi_event_scan_results` | `process_scan_results_event()` | Scan completed |
| `wifi_event_hal_wps_results` | `process_wps_results_event()` | WPS result |

### 4. Scheduler

**File**: `source/utils/scheduler.c`

Timer-based task scheduling.

| Function | Purpose |
|----------|---------|
| `scheduler_init()` | Initialize scheduler |
| `scheduler_add_timer_task()` | Add periodic/one-shot task |
| `scheduler_execute()` | Execute pending tasks |
| `scheduler_cancel_timer_task()` | Cancel a task |

**Key Scheduled Tasks** (from `ctrl_queue_timeout_scheduler_tasks`):
- `run_analytics_event()` - Analytics collection
- `run_greylist_event()` - RADIUS greylist check
- `sta_connectivity_selfheal()` - STA connection recovery
- `bus_check_and_subscribe_events()` - Bus event subscription retry
- `pending_states_webconfig_analyzer()` - Webconfig state analysis

---

## IPC Mechanisms

### 1. rbus (RDK-B Primary)

**File**: `source/platform/rdkb/bus.c` (~1900 lines)

| Handler Type | Function | Purpose |
|--------------|----------|---------|
| Get | `rbus_get_handler()` | Read parameter value |
| Set | `rbus_set_handler()` | Write parameter value |
| Method | `rbus_method_handler()` | RPC method call |
| Event | `rbus_event_sub_handler()` | Event subscription |
| Table Add | `rbus_table_add_row_handler()` | Add table row |
| Table Remove | `rbus_table_remove_row_handler()` | Remove table row |

**Registered Data Elements** (from `wifi_ctrl_rbus_handlers.c:3994`):
```
Device.WiFi.WebConfig.Data.Subdoc.North
Device.WiFi.WebConfig.Data.Subdoc.South
Device.WiFi.WebConfig.Data.Init
Device.WiFi.Private
Device.WiFi.Home
Device.WiFi.WiFiAPI.command
Device.WiFi.WiFiAPI.result
```

### 2. WebConfig Integration

**Directory**: `source/webconfig/` (45 files)

**Core File**: `wifi_webconfig.c`

| Function | Line | Purpose |
|----------|------|---------|
| `webconfig_init()` | 245 | Initialize all subdoc handlers |
| `webconfig_decode()` | 67 | Decode incoming subdoc |
| `webconfig_encode()` | 112 | Encode outgoing subdoc |

**Subdoc Types** (39 total):
| Type | File | Purpose |
|------|------|---------|
| `webconfig_subdoc_type_private` | `wifi_webconfig_private.c` | Private SSID |
| `webconfig_subdoc_type_home` | `wifi_webconfig_home.c` | Home network |
| `webconfig_subdoc_type_radio` | `wifi_webconfig_radio.c` | Radio config |
| `webconfig_subdoc_type_mesh` | `wifi_webconfig_mesh.c` | Mesh network |
| `webconfig_subdoc_type_mac_filter` | `wifi_webconfig_macfilter.c` | MAC filtering |
| `webconfig_subdoc_type_steering_config` | `wifi_webconfig_steering_config.c` | Band steering |

**Subdoc Handler Interface**:
```c
typedef struct {
    webconfig_subdoc_type_t type;
    webconfig_error_t (*init_subdoc)(webconfig_subdoc_t *doc);
    webconfig_error_t (*access_check)(webconfig_t *config, webconfig_subdoc_data_t *data);
    webconfig_error_t (*encode_subdoc)(webconfig_t *config, webconfig_subdoc_data_t *data);
    webconfig_error_t (*decode_subdoc)(webconfig_t *config, webconfig_subdoc_data_t *data);
    webconfig_error_t (*translate_to)(webconfig_t *config, webconfig_subdoc_data_t *data);
    webconfig_error_t (*translate_from)(webconfig_t *config, webconfig_subdoc_data_t *data);
} webconfig_subdoc_t;
```

### 3. TR-181 Data Model Layer

**File**: `source/dml/tr_181/ml/cosa_wifi_dml.c` (~21K lines)

Implements TR-181 Device.WiFi.* data model.

**Key Namespaces**:
```
Device.WiFi.RadioNumberOfEntries
Device.WiFi.Radio.{i}.*
Device.WiFi.SSID.{i}.*
Device.WiFi.AccessPoint.{i}.*
Device.WiFi.EndPoint.{i}.*
```

**CCSP Integration** (`source/dml/tr_181/sbapi/cosa_dbus_api.c`):
| Function | Purpose |
|----------|---------|
| `Cosa_Init()` | Initialize COSA with bus handle |
| `Cosa_GetParamValues()` | Get parameters via CcspBaseIf |
| `Cosa_SetParamValuesNoCommit()` | Set parameters |
| `Cosa_SetCommit()` | Commit parameter changes |

### 4. OVSDB (Persistent Storage)

**Directory**: `lib/ovsdb/`

**Database Location**: `/opt/secure/wifi/rdkb-wifi.db`

**OVSDB Tables** (14 tables):
| Table | Purpose |
|-------|---------|
| `Wifi_Radio_Config` | Radio parameters |
| `Wifi_VAP_Config` | VAP settings |
| `Wifi_Security_Config` | Security/authentication |
| `Wifi_Interworking_Config` | Hotspot 2.0 |
| `Wifi_GAS_Config` | GAS settings |
| `Wifi_Global_Config` | Global parameters |
| `Wifi_MacFilter_Config` | MAC filtering |
| `Wifi_Passpoint_Config` | Passpoint |
| `Wifi_Anqp_Config` | ANQP |
| `Wifi_Preassoc_Control_Config` | Pre-association control |
| `Wifi_Postassoc_Control_Config` | Post-association control |
| `Wifi_Rfc_Config` | RFC feature flags |

**Key Functions** (`source/db/wifi_db_apis.c`):
```c
bool wifidb_update_wifi_radio_config(wifi_radio_operationParam_t *config);
bool wifidb_update_wifi_vap_info(wifi_vap_info_t *vap_info);
bool wifidb_update_wifi_security_config(wifi_vap_security_t *sec);
bool wifidb_get_wifi_radio_config(int radio_index, wifi_radio_operationParam_t *config);
```

---

## Data Flow

### Configuration Push (WebConfig)

```
Cloud/TR-181 sends JSON
    │
    ▼
rbus callback receives data
    │
    ▼
push_event_to_ctrl_queue(wifi_event_type_webconfig, wifi_event_webconfig_set_data)
    │
    ▼
ctrl_queue_loop() wakes up
    │
    ▼
handle_webconfig_event()  [wifi_ctrl_queue_handlers.c:4317]
    │
    ├─► webconfig_decode()  [wifi_webconfig.c:67]
    │       │
    │       ├─► find_subdoc_type()
    │       ├─► doc->decode_subdoc()  (JSON → struct)
    │       └─► doc->translate_from_subdoc()
    │
    ├─► Apply to HAL (e.g., webconfig_hal_radio_apply())
    │
    ├─► Update OVSDB (wifidb_update_*)
    │
    └─► apps_mgr_event()  (notify apps)
```

### Client Connect Event

```
WiFi HAL: Client associates
    │
    ▼
wifi_hal_newApAssociatedDevice_callback()
    │
    ▼
device_associated()  [wifi_ctrl.c]
    │
    ▼
push_event_to_ctrl_queue(wifi_event_type_hal_ind, wifi_event_hal_assoc_device)
    │
    ▼
ctrl_queue_loop() wakes up
    │
    ▼
handle_hal_indication()  [wifi_ctrl_queue_handlers.c:4168]
    │
    ▼
process_assoc_device_event()  [wifi_ctrl_queue_handlers.c:2354]
    │
    ├─► hash_map_put() to associated_devices_map
    ├─► notify_associated_entries() (rbus notification)
    ├─► notify_hotspot() (if hotspot VAP)
    └─► assoc_dev_notify_LM_lite()
```

### Radio Configuration Change

```
webconfig_hal_radio_apply()  [wifi_ctrl_webconfig.c:2266]
    │
    ├─► is_radio_param_config_changed()  (detect changes)
    │
    ├─► wifi_radio_operationParam_validation()  (validate)
    │
    ├─► wifi_hal_setRadioOperatingParameters()  (apply to HAL)
    │
    ├─► wifidb_update_wifi_radio_config()  (persist to OVSDB)
    │
    └─► start_wifi_sched_timer()  (schedule followup)
```

---

## State Machines

### 1. WebConfig State Machine

**Location**: `source/core/wifi_ctrl.h`

Tracks pending configuration responses using bitmask:

```c
typedef enum {
    ctrl_webconfig_state_none                           = 0,
    ctrl_webconfig_state_radio_cfg_rsp_pending          = 0x0001,
    ctrl_webconfig_state_vap_all_cfg_rsp_pending        = 0x0002,
    ctrl_webconfig_state_vap_private_cfg_rsp_pending    = 0x0004,
    ctrl_webconfig_state_vap_home_cfg_rsp_pending       = 0x0008,
    ctrl_webconfig_state_vap_xfinity_cfg_rsp_pending    = 0x0010,
    ctrl_webconfig_state_vap_mesh_cfg_rsp_pending       = 0x0020,
    ctrl_webconfig_state_macfilter_cfg_rsp_pending      = 0x0040,
    ctrl_webconfig_state_factoryreset_rsp_pending       = 0x0080,
    ctrl_webconfig_state_sta_conn_status_rsp_pending    = 0x0100,
    ctrl_webconfig_state_associated_clients_rsp_pending = 0x0200,
    // ... more states ...
    ctrl_webconfig_state_max                            = 0x10000000
} wifi_ctrl_webconfig_state_t;
```

**Usage**: `ctrl->webconfig_state |= ctrl_webconfig_state_radio_cfg_rsp_pending;`

### 2. STA Connection State Machine (Mesh Extender)

**Location**: `source/core/services/vap_svc.h`

For mesh extender STA connection management:

```c
typedef enum {
    connection_state_disconnected_scan_list_none,
    connection_state_disconnected_scan_list_in_progress,
    connection_state_disconnected_scan_list_all,
    connection_state_disconnected_steady,
    connection_state_connection_in_progress,
    connection_state_connection_to_lcb_in_progress,
    connection_state_connection_to_nb_in_progress,
    connection_state_connected,
    connection_state_connected_wait_for_csa,
    connection_state_connected_scan_list,
    connection_state_disconnection_in_progress,
} connection_state_t;
```

**State Transitions**:
```
disconnected_scan_list_none
    │ (start scan)
    ▼
disconnected_scan_list_in_progress
    │ (scan complete)
    ▼
disconnected_scan_list_all
    │ (select best AP)
    ▼
connection_in_progress
    │ (auth/assoc success)
    ▼
connected
    │ (CSA received)
    ▼
connected_wait_for_csa
```

---

## Apps Manager

**File**: `source/apps/wifi_apps_mgr.c`

Forwards events to registered application modules.

| Function | Line | Purpose |
|----------|------|---------|
| `apps_mgr_init()` | 45 | Initialize all apps |
| `apps_mgr_event()` | 103 | Forward event to apps |
| `apps_mgr_analytics_event()` | 150 | Analytics-specific events |

**Registered Apps**:
| App | Directory | Purpose |
|-----|-----------|---------|
| Analytics | `source/apps/analytics/` | Telemetry collection |
| Blaster | `source/apps/blaster/` | WiFi performance testing |
| CSI | `source/apps/csi/` | Channel State Information |
| Harvester | `source/apps/harvester/` | Data harvesting |
| SM | `source/apps/sm/` | Statistics Manager |
| EM | `source/apps/em/` | EasyMesh |
| CAC | `source/apps/cac/` | Connection Admission Control |
| LEVL | `source/apps/levl/` | LEVL feature |
| Motion | `source/apps/motion/` | Motion detection |
| STA Manager | `source/apps/sta_mgr/` | Station management |

---

## WiFi HAL Integration

### Callback Registration

**File**: `source/core/wifi_ctrl.c` (in `init_wifi_ctrl()`)

```c
// Client events
wifi_hal_newApAssociatedDevice_callback_register(device_associated);
wifi_hal_apDeAuthEvent_callback_register(device_deauthenticated);
wifi_hal_apDisassociatedDevice_callback_register(device_disassociated);

// STA mode
wifi_hal_staConnectionStatus_callback_register(sta_connection_status);

// Scanning
wifi_hal_scanResults_callback_register(scan_results_callback);

// Security
wifi_hal_radius_eap_failure_callback_register(radius_eap_failure_callback);
wifi_hal_handshake_callback_register(handle_handshake_status);

// Management frames
wifi_hal_mgmt_frame_callbacks_register(mgmt_wifi_frame_recv);

// Channel events
wifi_chan_event_register(channel_change_callback);

// WPS
wifi_wpsEvent_callback_register(wps_event_callback);
```

### Key HAL Operations

| Operation | HAL Function |
|-----------|--------------|
| Create VAP | `wifi_hal_createVAP()` |
| Set radio params | `wifi_hal_setRadioOperatingParameters()` |
| Start scan | `wifi_hal_startScan()` |
| Connect (STA) | `wifi_hal_connect()` |
| Disconnect | `wifi_hal_disconnect()` |
| Get capabilities | `wifi_hal_getHalCapability()` |

---

## Critical Data Structures

### wifi_mgr_t (Global Manager)

```c
typedef struct {
    wifi_hal_capability_t    hal_cap;      // HAL capabilities
    wifi_global_param_t      global_param; // Global config
    wifi_radio_config_t      radio_config[MAX_NUM_RADIOS];
    wifi_ctrl_t              ctrl;         // Controller state
    wifi_db_t                *db;          // Database handle
    wifi_apps_mgr_t          apps_mgr;     // Apps manager
} wifi_mgr_t;
```

### wifi_ctrl_t (Controller)

```c
typedef struct {
    pthread_t                thread_id;
    pthread_mutex_t          lock;
    pthread_cond_t           cond;
    queue_t                  *queue;        // Event queue
    scheduler_t              *scheduler;    // Timer scheduler
    webconfig_t              webconfig;     // WebConfig handler
    wifi_ctrl_webconfig_state_t webconfig_state;  // State machine
    hash_map_t               *vap_map;      // VAP hash map
} wifi_ctrl_t;
```

---

## Critical Analysis

### Tight Coupling Areas

1. **wifi_ctrl_queue_handlers.c** (4500+ lines)
   - Handles all event types in one file
   - Recommendation: Split by event category

2. **HAL callbacks directly modify control state**
   - Tightly coupled to wifi_ctrl internals
   - Recommendation: Add callback abstraction layer

### Hardcoded Configurations

| Config | Value | Location |
|--------|-------|----------|
| Database path | `/opt/secure/wifi/rdkb-wifi.db` | wifi_db_apis.c |
| Log path prefix | `/nvram/` | wifi_util.c |
| Schema version | `100007` | wifi_db_apis.c |
| Poll timeout | 200ms | wifi_ctrl.c |

### Potential Race Conditions

1. **associated_devices_map access**
   - Protected by `associated_devices_lock` mutex
   - Verify all access paths use lock

2. **Event queue push/pop**
   - Uses pthread condition variables
   - Generally safe but verify signal handling

### Refactoring Opportunities

1. Split `wifi_ctrl_queue_handlers.c` by event type
2. Extract HAL callback registration to dedicated module
3. Add event handler plugin architecture
4. Create configuration abstraction layer
5. Improve error propagation with result types
