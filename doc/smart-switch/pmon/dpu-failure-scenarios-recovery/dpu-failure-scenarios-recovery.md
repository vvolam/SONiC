# Smart Switch: PMON: DPU Failure Scenarios and Recovery HLD #

## Table of Content ##

- [Revision](#revision)
- [Scope](#scope)
- [Definitions/Abbreviations](#definitionsabbreviations)
- [Overview](#overview)
- [Quick Scenario Summary](#quick-scenario-summary)
- [PMON Critical Processes](#pmon-critical-processes)
- [Timers and Thresholds](#timers-and-thresholds)
- [Terminology](#terminology)
- [DPU Ready/Reset Status DB Info](#dpu-readyreset-status-db-info)
- [DPU Software Failures](#dpu-software-failures)
  - [Critical Process Crash](#critical-process-crash)
  - [Critical Process Restart on DPU](#critical-process-restart-on-dpu)
  - [Critical Process Persistently Down on DPU](#critical-process-persistently-down-on-dpu)
  - [pmon Crash on NPU](#pmon-crash-on-npu)
  - [databasedpu Crash on NPU](#databasedpu-crash-on-npu)
- [DPU Hardware Failures](#dpu-hardware-failures)
  - [DPU Hardware Failure (Complete DPU Down)](#dpu-hardware-failure-complete-dpu-down)
  - [DPU Power Failure / Unexpected Shutdown](#dpu-power-failure--unexpected-shutdown)
  - [PCIe Failure](#pcie-failure)
- [NPU / Switch Level Failures](#npu--switch-level-failures)
  - [NPU Kernel Crash / Memory Exhaustion](#npu-kernel-crash--memory-exhaustion)
- [Planned Operations](#planned-operations)
  - [DPU Graceful Shutdown](#dpu-graceful-shutdown)
  - [DPU Cold Reboot](#dpu-cold-reboot)
  - [Full SmartSwitch Reboot](#full-smartswitch-reboot)
- [Scenario DB State Summary](#scenario-db-state-summary)
- [Syslog Signatures and Alerting](#syslog-signatures-and-alerting)
- [Platform Behavior: NPU/DPU Reboot Standardization](#platform-behavior-npudpu-reboot-standardization)
- [References](#references)

---

## Revision ##

|  Rev  |        Author       | Change Description                     |
| :---: |  :----------------: | -------------------------------------- |
|  0.1  |  Vasundhara Volam   | Initial Version                        |

---

## Scope ##

This document covers the High Level Design for DPU failure scenarios on a SmartSwitch from the PMON (Platform Monitor) perspective — specifically focused on detection, DB state management, and recovery actions performed by `chassisd` and other PMON sub-daemons.

The scope includes:

- DPU software failures (critical process crashes and restarts on DPU; pmon and databasedpu crashes on NPU)
- DPU hardware failures (complete DPU down, power failure / unexpected shutdown, PCIe failure)
- NPU/switch-level failures (kernel crash, memory exhaustion)
- Planned operations (DPU graceful shutdown, DPU cold reboot, full SmartSwitch reboot)
- DB state tracking for DPU failure detection and recovery (new and existing DB entries)
- PMON critical process definitions and criticality levels
- Timers and thresholds used by PMON for failure detection and recovery
- Syslog signatures for chronic failures and alerting
- Platform behavior standardization for NPU/DPU reboots

---

## Definitions/Abbreviations ##

| Term | Meaning                                                 |
| ---- | ------------------------------------------------------- |
| NPU  | Network Processing Unit                                 |
| DPU  | Data Processing Unit                                    |
| PCIe | PCI Express (Peripheral Component Interconnect Express) |
| gNOI | gRPC Network Operations Interface                       |
---

## Overview ##

SmartSwitch consists of one NPU (switch ASIC) and multiple DPUs. All front panel ports are connected to the NPU. DPUs are connected to the NPU via PCIe and back-panel ports.

The PMON (Platform Monitor) daemon on the NPU is responsible for monitoring DPU health and managing DPU lifecycle operations. Its primary sub-daemon, `chassisd`, continuously polls DPU states (midplane, control plane, data plane), detects failures, performs recovery actions (power-cycle, PCIe rescan), and updates database entries to reflect DPU readiness.

This document enumerates all failure scenarios that can occur on a DPU or its supporting infrastructure from the PMON perspective, describes detection mechanisms driven by `chassisd`, recovery paths, and the corresponding database state changes. It also covers planned operations (graceful shutdown, cold reboot, full SmartSwitch reboot) and the DB state changes introduced to support them.

---

## Quick Scenario Summary ##

| Failure Scenario | Detection (by PMON) | PMON Action | DB Transitions |
| ---------------- | ------------------- | ----------- | -------------- |
| Critical process restart on DPU | `chassisd` polls `dpu_control_plane_state` | Set `dpu_healthy=false`, `reset_status=true`; wait for auto-recovery | `dpu_healthy`: `true` → `false` → `true`; `reset_status`: `false` → `true` |
| Critical process persistently down on DPU | `chassisd` polls `dpu_control_plane_state`; no recovery within timeout | Power-cycle DPU; increment `reset_count` | `dpu_healthy`: `true` → `false` → `true`; `reset_status`: `false` → `true` |
| pmon crash on NPU | Health updates to `CHASSIS_STATE_DB` stop | On restart: set `reset_status=true`, `dpu_healthy=false` for all DPUs; re-poll | `dpu_healthy`: unknown → `false` → `true`; `reset_status`: unknown → `true` |
| databasedpu crash on NPU | DPU state reads fail | Set `dpu_healthy=false`; wait for `systemd` restart | `dpu_healthy`: `true` → `false` → `true` |
| DPU hardware failure (complete DPU down) | `CHASSIS_MODULE_TABLE` oper_status → `offline` | Power-cycle DPU; increment `reset_count` | `dpu_healthy`: `true` → `false` → `true`; `reset_status`: `false` → `true` |
| DPU power failure / unexpected shutdown | Midplane ping failure; `dpu_midplane_link_state` → `down` | Power-cycle DPU; increment `reset_count` | `dpu_healthy`: `true` → `false` → `true`; `reset_status`: `false` → `true` |
| PCIe failure | `pmon` PCIe daemon; `dpu_midplane_link_state` → `down` | Power-cycle DPU; PCIe rescan; increment `reset_count` | `dpu_healthy`: `true` → `false` → `true`; `PCIE_DETACH_INFO`: `detached` → `reattached` |
| NPU kernel crash / memory exhaustion | `chassisd` detects stale DPU states on recovery | Re-poll DPUs; gNOI reboot DPUs | `dpu_healthy`: unknown → `false` → `true`; `reset_status`: unknown → `true` |
| DPU graceful shutdown | N/A (planned) | PCIe detach + Controlled shutdown via gNOI; power-off; PCIe detach | `dpu_healthy`: `true` → `false`; `reset_status`: → `true` |
| DPU cold reboot | N/A (planned) | PCIe detach + gNOI halt + power-cycle + PCIe reattach | `dpu_healthy`: `true` → `false` → `true`; `PCIE_DETACH_INFO`: transitions; `reset_status`: → `true` |
| Full SmartSwitch reboot | N/A (planned) | gNOI halt all DPUs; NPU reboot; power-cycle all DPUs | `dpu_healthy`: `true` → `false` → `true` per DPU; `reset_status`: → `true` |

---

## PMON Critical Processes ##

The following processes are considered critical for SmartSwitch operation from the PMON perspective. A failure in any of these impacts PMON's ability to monitor and manage DPUs.

**PMON-managed processes (on NPU):**

| Process | Criticality | Role | Failure Impact |
| ------- | ----------- | ---- | -------------- |
| `chassisd` | **Critical** | Monitors DPU health (midplane, control plane, data plane); manages power-cycle, reset, and DB state updates | All DPU failure detection and recovery stops; no DB updates for `dpu_healthy`, `reset_status` |
| `pcied` | **Critical** | Monitors PCIe link state between NPU and DPUs | PCIe failures go undetected; `PCIE_DETACH_INFO` not updated |
| `thermalctld` | High | Monitors thermal sensors on NPU and DPUs | Thermal events may not trigger DPU shutdown; risk of hardware damage |
| `sensord` | High | Reads hardware sensor data (voltage, fan, temperature) | Sensor-based alerts and thresholds not enforced |
| `gnoi_reboot_daemon.py` | **Critical** | Sends gNOI Reboot RPCs to DPUs for graceful shutdown / reboot | Graceful shutdown and planned reboot operations fail |

**DPU-side processes (monitored indirectly by PMON):**

| Process | Criticality | PMON Visibility | Failure Effect on PMON |
| ------- | ----------- | --------------- | ---------------------- |
| `syncd` | **Critical** | `dpu_control_plane_state` → `down` | `chassisd` detects via polling; triggers reset flow |
| `swss` | **Critical** | `dpu_control_plane_state` → `down` | `chassisd` detects via polling; triggers reset flow |
| `database` | **Critical** | `dpu_control_plane_state` → `down` | `chassisd` detects via polling; triggers reset flow |
| `pmon` (on DPU) | High | `dpu_control_plane_state` and `dpu_data_plane_state` updates stop | `chassisd` may see stale states; indirect detection via midplane polling |

---

## Timers and Thresholds ##

All timers and thresholds used by PMON for DPU failure detection and recovery are listed below. Values shown are defaults; some are configurable via `platform.json`.

| Timer / Threshold | Default Value | Configurable | Used By | Description |
| ----------------- | :-----------: | :----------: | ------- | ----------- |
| `chassisd` health poll interval | 10 seconds | No | `chassisd` | Interval at which `chassisd` polls `dpu_control_plane_state`, `dpu_data_plane_state`, and `dpu_midplane_link_state` |
| DPU auto-recovery timeout | 30 seconds | Yes | `chassisd` | Time allowed for a DPU to recover from a critical process restart before escalating |
| DPU power-cycle timeout | 180 seconds | Yes | `chassisd` | Time `chassisd` waits for `dpu_control_plane_state` to return to `up` before issuing a power-cycle |
| `dpu_halt_services_timeout` | 300 seconds | Yes (`platform.json`) | `gnoi_reboot_daemon.py` | Maximum time to wait for DPU services to halt gracefully during reboot/shutdown |
| `reset_limit` | Platform-defined | Yes (`platform.json`) | `chassisd` | Maximum number of power-cycle attempts before marking DPU as unrecoverable |

---

## Terminology ##

| Term | Explanation |
| ---- | ----------- |
| chassisd | Chassis daemon running inside `pmon` on the NPU; monitors DPU health states, manages DPU power-cycle and reset operations |
| pmon | Platform Monitor daemon on NPU; hosts `chassisd` and other hardware monitoring sub-daemons |
| syncd | Sync daemon; manages SAI API calls to DPU ASIC |
| control plane state | DPU SONiC is booted up, all containers are up, interfaces are up, and DPU is ready to accept configuration. Derived from SYSTEM_READY in STATE_DB. Values: `"unknown"`, `"up"`, `"down"`. |
| midplane link state | The PCIe link between the NPU and DPU is operational. Monitored and updated by NPU pmon `chassisd` via the `is_midplane_reachable` platform API. Values: `"unknown"`, `"up"`, `"down"`. |
| dataplane state | Configuration is downloaded, pipeline stages are up, and DPU hardware (port/ASIC) is ready to take traffic. Values: `"unknown"`, `"up"`, `"down"`. |

---

## DPU Ready/Reset Status DB Info ##

### Existing DB entries ###

The following DB entries track the DPU lifecycle state and are referenced during failure detection and recovery.

**DPU State in CHASSIS_STATE_DB:**

```
DPU_STATE|DPU<dpu_index>:
{
  "dpu_control_plane_state": "up" | "down",
  "dpu_control_plane_time":  "<UTC timestamp>",
  "dpu_data_plane_state":    "up" | "down",
  "dpu_data_plane_time":      "<UTC timestamp>",
  "dpu_midplane_link_state": "up" | "down",
  "dpu_midplane_link_time":      "<UTC timestamp>"
}
```

**PCIe Detach Info in STATE_DB:**

```
PCIE_DETACH_INFO|DPU<dpu_index>:
{
  "dpu_id":    "<index>",
  "dpu_state": "detaching" | "detached" | "reattached",
  "bus_info":  "[DDDD:]BB:SS.F"
}
```

**Graceful Shutdown / Reboot Tracking in STATE_DB:**

```
CHASSIS_MODULE_TABLE|DPU<dpu_index>:
{
  "oper_status":                  "Online" | "Offline",
  "state_transition_in_progress": "True" | "False",
  "transition_start_time":        "<UTC timestamp>",
  "transition_type":              "shutdown" | "reboot" | "none"
}
```

### New DB entries ###

The following DB entries will now be newly created to track DPU failure states.

**DPU Reset Info in CHASSIS_STATE_DB on NPU**

```
DPU_RESET_INFO_TABLE|DPU<dpu_index>:
{
  "reset_status":  "true" | "false",
  "reset_count":   "<integer>",
  "timestamp":     "<UTC timestamp>",
  "reset_limit":   "<integer>"
}
```

| Field | Description | Set by | Cleared by |
| ----- | ----------- | ------ | ---------- |
| `reset_status` | Set to `"true"` when a DPU undergoes a reset, indicating that DPU rules need to be reprogrammed | `chassisd` | External controller (set to `"false"` after reprogramming) |
| `reset_count` | Number of times the DPU has been reset | `chassisd` (incremented on each power-cycle) | — |
| `timestamp` | UTC timestamp of the last DPU reset | `chassisd` | — |
| `reset_limit` | Maximum number of resets before marking the DPU as unrecoverable | Configuration / platform default | — |

---

**dpu_healthy field in CHASSIS_MODULE_TABLE in STATE_DB**

```
CHASSIS_MODULE_TABLE|DPU<dpu_index>:
{
  "dpu_healthy":    "true" | "false"
}
```

| Field | Description | Set by | Cleared by |
| ----- | ----------- | ------ | ---------- |
| `dpu_healthy` | Indicates whether the DPU is ready for rules to be programmed. Set to `"true"` when DPU midplane, control plane, and data plane are all up and operational. Set to `"false"` otherwise. | `chassisd` (set to `"true"`) | `chassisd` (set to `"false"` on failure/reset) |

---

## DPU Software Failures ##

### Critical process list ###

A critical process list covers any essential DPU or NPU process failing unexpectedly during operation. The behavior depends on which process crashed and whether PMON can detect and act on it.

**Processes and PMON impact:**

| Process | Location | PMON-Managed | Impact from PMON Perspective |
| ------- | -------- | :----------: | ---------------------------- |
| syncd | DPU | No (detected) | `dpu_control_plane_state` → `down`; `chassisd` triggers reset flow |
| swss | DPU | No (detected) | `dpu_control_plane_state` → `down`; `chassisd` triggers reset flow |
| database | DPU | No (detected) | `dpu_control_plane_state` → `down`; `chassisd` triggers reset flow |
| pmon | DPU | No (detected) | State updates to NPU stop; `chassisd` may see stale states |
| pmon | NPU | Yes | DPU health signals lost; all `chassisd` monitoring stops |
| databasedpu\<dpu-index\> | NPU | No (`systemd`) | `chassisd` cannot read DPU state from Redis |

Each is detailed in the subsections below.

---

### Critical process restart on DPU ###

**Description:**
When any process in the `syncd` or `swss` dockers crashes on the DPU, but the container supervisor successfully restarts the process and the DPU recovers on its own within the auto-recovery timeout (default: 30 seconds). No power-cycle is needed.

**Detection (by PMON):**
- `chassisd` on the NPU polls `dpu_control_plane_state` every 10 seconds and observes it as `down`.

**PMON Action:**
- `chassisd` sets `reset_status` to `true` and `dpu_healthy` to `false` for the corresponding DPU.
- `chassisd` waits up to 30 seconds for the DPU to self-recover.
- Once `dpu_control_plane_state` transitions back to `up`, `chassisd` verifies all DPU states (midplane, control plane, data plane) and sets `dpu_healthy` back to `true`.

**DB State Transition:**

| DB Field | Before | During Failure | After Recovery |
| -------- | :----: | :------------: | :------------: |
| `dpu_control_plane_state` | `up` | `down` | `up` |
| `dpu_healthy` | `true` | `false` | `true` |
| `reset_status` | `false` | `true` | `true` (cleared by external controller) |

**External State (outside PMON):**
- External controller reprograms DPU rules and clears `reset_status` to `false`.

---

### Critical process persistently down on DPU ###

**Description:**
When any critical process in `syncd`, `swss`, `pmon`, or `database` crashes on the DPU and **remains down beyond the auto-recovery timeout** (i.e., the container supervisor cannot successfully restart it, or the process keeps crash-looping). Unlike a transient restart, this scenario indicates a persistent failure that requires a DPU power-cycle to recover.

**Detection (by PMON):**
- `chassisd` on the NPU polls `dpu_control_plane_state` every 10 seconds and observes it as `down`.
- State remains `down` beyond the 30-second auto-recovery timeout.

**PMON Action:**
- `chassisd` sets `reset_status` to `true` and `dpu_healthy` to `false` for the corresponding DPU.
- After the power-cycle timeout (default: 180 seconds) elapses without recovery, `chassisd` issues a power-cycle of the DPU and increments `reset_count`.
- Once `dpu_control_plane_state` transitions back to `up`, `chassisd` verifies all DPU states (midplane, control plane, data plane) and sets `dpu_healthy` back to `true`.
- If `reset_count` reaches `reset_limit`, `chassisd` marks the DPU as unrecoverable (see [Syslog Signatures and Alerting](#syslog-signatures-and-alerting)).

**DB State Transition:**

| DB Field | Before | During Failure | After Recovery |
| -------- | :----: | :------------: | :------------: |
| `dpu_control_plane_state` | `up` | `down` | `up` |
| `dpu_healthy` | `true` | `false` | `true` |
| `reset_status` | `false` | `true` | `true` (cleared by external controller) |
| `reset_count` | N | N | N+1 |

**External State (outside PMON):**
- External controller reprograms DPU rules and clears `reset_status` to `false`.

---

### pmon crash on NPU ###

**Description:**
The `pmon` (Platform Monitor) daemon on the NPU crashes. This is a **critical** PMON failure — `chassisd` and all other PMON sub-daemons stop, halting all DPU health monitoring.

**Detection (by PMON):**
- Not self-detectable. `systemd` detects the `pmon` container is down and restarts it.
- DPU health state updates to `CHASSIS_STATE_DB` stop during the outage.

**PMON Action:**
- On `chassisd` bringup sequence after restart, `chassisd` sets `reset_status` to `true` and `dpu_healthy` to `false` for **all** DPUs.
- `chassisd` re-polls all DPU states and updates `CHASSIS_STATE_DB` with current values.
- For each DPU found healthy, `chassisd` sets `dpu_healthy` back to `true`.

**DB State Transition:**

| DB Field | Before | During Failure | After Recovery |
| -------- | :----: | :------------: | :------------: |
| `dpu_healthy` (all DPUs) | `true` | stale | `false` → `true` (per DPU) |
| `reset_status` (all DPUs) | varies | stale | `true` |

**External State (outside PMON):**
- `systemd` restarts the `pmon` container.
- Any DPU state changes during the outage are reconciled on restart.

---

### databasedpu crash on NPU ###

**Description:**
The `databasedpu<dpu-index>` (per-DPU Redis database instance) on the NPU crashes. Each DPU has a dedicated Redis instance on the NPU (port 6381 + DPU ID, bound to midplane bridge IP 169.254.200.254).

**Detection (by PMON):**
- `chassisd` cannot read DPU state from the corresponding Redis instance.

**PMON Action:**
- `chassisd` detects loss of DPU state and sets `dpu_healthy` to `false`.
- After `systemd` restarts the Redis instance and DPU reconnects, `chassisd` polls DPU state and sets `dpu_healthy` back to `true` once all states are verified.

**DB State Transition:**

| DB Field | Before | During Failure | After Recovery |
| -------- | :----: | :------------: | :------------: |
| `dpu_healthy` | `true` | `false` | `true` |

**External State (outside PMON):**
- `systemd` restarts `databasedpu<dpu-index>`.
- Redis instance recovers from RDB/AOF persistence files.
- DPU reconnects to its remote Redis instance via midplane bridge IP:port.

---

## DPU Hardware Failures ##

### DPU Hardware Failure (Complete DPU Down) ###

**Description:**
A DPU completely fails due to hardware fault, thermal event, or unrecoverable error. The DPU is no longer responsive on the midplane or back-panel ports.

**Detection (by PMON):**
- NPU: Oper state of the DPU `CHASSIS_MODULE_TABLE|DPU<dpu_index>|oper_status` is set to `offline`.

**PMON Action:**
- `chassisd` sets `reset_status` to `true` and `dpu_healthy` to `false`.
- `chassisd` power-cycles the DPU and increments `reset_count`.
- After power-cycle, DPU goes through full boot sequence: midplane attach → PCIe rescan → SONiC boot → container startup.
- `chassisd` verifies all DPU states (midplane, control plane, data plane) and sets `dpu_healthy` back to `true`.
- If `reset_count` reaches `reset_limit`, `chassisd` marks the DPU as unrecoverable (see [Syslog Signatures and Alerting](#syslog-signatures-and-alerting)).

**DB State Transition:**

| DB Field | Before | During Failure | After Recovery |
| -------- | :----: | :------------: | :------------: |
| `oper_status` | `Online` | `Offline` | `Online` |
| `dpu_healthy` | `true` | `false` | `true` |
| `reset_status` | `false` | `true` | `true` (cleared by external controller) |
| `reset_count` | N | N | N+1 |

**External State (outside PMON):**
- External controller reprograms DPU rules and clears `reset_status` to `false`.

---

### DPU Power Failure / Unexpected Shutdown ###

**Description:**
The DPU loses power unexpectedly or shuts down without graceful notification (e.g., voltage regulator failure, firmware crash).

**Detection (by PMON):**
- NPU `pmon` detects midplane ping failure → `dpu_midplane_link_state` set to `down`.
- `dpu_control_plane_state` transitions to `down`.

**PMON Action:**
- `chassisd` sets `reset_status` to `true` and `dpu_healthy` to `false`.
- `chassisd` power-cycles the DPU and increments `reset_count`.
- After power-cycle, `chassisd` verifies all DPU states and sets `dpu_healthy` back to `true`.

**DB State Transition:**

| DB Field | Before | During Failure | After Recovery |
| -------- | :----: | :------------: | :------------: |
| `dpu_midplane_link_state` | `up` | `down` | `up` |
| `dpu_control_plane_state` | `up` | `down` | `up` |
| `dpu_healthy` | `true` | `false` | `true` |
| `reset_status` | `false` | `true` | `true` (cleared by external controller) |
| `reset_count` | N | N | N+1 |

**External State (outside PMON):**
- External controller reprograms DPU rules and clears `reset_status` to `false`.

---

### PCIe Failure ###

**Description:**
The PCIe bus between the NPU and a local DPU fails, making the DPU unreachable from the NPU. The DPU may still be running internally but is disconnected from the NPU.

**Detection (by PMON):**
- `pmon` PCIe daemon (`pcied`) detects PCIe link down.
- `CHASSIS_STATE_DB` updated: `dpu_midplane_link_state` → `down`.
- `PCIE_DETACH_INFO|DPU<dpu_index>` updated with `dpu_state: detached`.

**PMON Action:**
- `chassisd` sets `reset_status` to `true` and `dpu_healthy` to `false`.
- `chassisd` power-cycles the DPU and increments `reset_count`.
- After power-cycle, PCIe rescan is performed:
  - Vendor API: `pci_reattach()`
  - Or sysfs: `echo 1 > /sys/bus/pci/rescan`
- `chassisd` verifies all DPU states (midplane, control plane, data plane) and sets `dpu_healthy` back to `true`.
- Timeout for DPU halt services: `dpu_halt_services_timeout` (default: 300 seconds, configurable in `platform.json`).

**DB State Transition:**

| DB Field | Before | During Failure | After Recovery |
| -------- | :----: | :------------: | :------------: |
| `dpu_midplane_link_state` | `up` | `down` | `up` |
| `PCIE_DETACH_INFO` `dpu_state` | `reattached` | `detached` | `reattached` |
| `dpu_healthy` | `true` | `false` | `true` |
| `reset_status` | `false` | `true` | `true` (cleared by external controller) |
| `reset_count` | N | N | N+1 |

**External State (outside PMON):**
- External controller reprograms DPU rules and clears `reset_status` to `false`.

---

## NPU / Switch Level Failures ##

### NPU Kernel Crash / Memory Exhaustion ###

**Description:**
The entire switch (NPU + all DPUs) goes down due to kernel panic or memory exhaustion. All DPUs on the switch are impacted simultaneously.

**Detection (by PMON):**
- On NPU recovery, `chassisd` detects DPU states may be stale (DB was lost during crash).

**PMON Action:**
- On recovery, `chassisd` initializes all DPU states as `down` and sets `reset_status` to `true`, `dpu_healthy` to `false` for all DPUs.
- `chassisd` re-establishes midplane connectivity and polls each DPU's state.
- If a DPU is still running and healthy (midplane, control plane, data plane all `up`), `chassisd` sets `dpu_healthy` back to `true`.
- If a DPU is unresponsive or in a bad state, `chassisd` sends gNOI Reboot RPC to reset it. Each such DPU then goes through: midplane attach → PCIe rescan → SONiC boot → container startup.

**DB State Transition:**

| DB Field | Before Crash | On NPU Recovery | After DPU Recovery |
| -------- | :----------: | :-------------: | :----------------: |
| `dpu_healthy` (all DPUs) | `true` | `false` | `true` (per DPU) |
| `reset_status` (all DPUs) | varies | `true` | `true` (cleared by external controller) |

**External State (outside PMON):**
- Only the NPU reboots; DPUs are not automatically power-cycled by the kernel crash itself.
- External controller reprograms all DPU rules after recovery.

---

## Planned Operations ##

### DPU Graceful Shutdown ###

**Description:**
Orderly shutdown of a DPU via CLI command: `config chassis module shutdown DPU<x>`.

**PMON Sequence:**
1. `chassisd` calls `set_admin_state(down)` → `module_base.py` triggers `graceful_shutdown_handler()`.
2. `CHASSIS_MODULE_INFO_TABLE` in STATE_DB updated:
   - `state_transition_in_progress`: `True`
   - `transition_start_time`: `<UTC timestamp>`
   - `transition_type`: `shutdown`
3. `chassisd` updates CHASSIS_STATE_DB:
   - `DPU_RESET_INFO_TABLE|DPU<dpu_index>`: `reset_status`: `true`
4. `chassisd` updates STATE_DB:
   - `CHASSIS_MODULE_TABLE|DPU<dpu_index>`: `dpu_healthy`: `false`
5. `gnoi_reboot_daemon.py` detects the transition and sends gNOI Reboot RPC (Method: `HALT`) to DPU.
6. DPU gracefully shuts down all services via `reboot -p`.
7. NPU polls `gnoi_client -rpc RebootStatus` until `active=false` (services terminated).
8. `state_transition_in_progress` set to `False`.
9. `module_base.py` calls platform API `power_down()` to power off DPU.
10. PCIe detach: `pci_detach()` or `echo 1 > /sys/bus/pci/devices/XXXX:XX:XX.X/remove`.
11. Sensor ignore configs added, sensord restarted.

**DB State Transition:**

| DB Field | Before | After Shutdown |
| -------- | :----: | :------------: |
| `dpu_healthy` | `true` | `false` |
| `reset_status` | `false` | `true` |
| `oper_status` | `Online` | `Offline` |
| `state_transition_in_progress` | `False` | `True` → `False` |

**Race Condition Handling:**
- If module shutdown is requested during a DPU reboot: operation fails; retry after reboot completes.
- If switch reboot is requested during module shutdown: graceful shutdown completes; switch reboot proceeds.
- Concurrent startup/shutdown on the same module: fails; user retries later.

---

### DPU Cold Reboot ###

**Description:**
Reboot a DPU with full power-cycle via CLI: `reboot -d <DPU_ID>`.

**PMON Sequence:**
1. NPU sends gNOI Reboot RPC (Method: `HALT`) to DPU.
2. NPU polls gNOI `RebootStatus` until `active=false` and `Status=STATUS_SUCCESS`.
3. Timeout: `dpu_halt_services_timeout` (default from `platform.json`, typically 300 seconds).
4. PCIe detach: `pci_detach()` or sysfs `echo 1 > /sys/bus/pci/devices/XXXX:XX:XX.X/remove`.
5. Platform vendor reboot API invoked (DPU cold boot / power-cycle).
6. PCIe reattach: `pci_reattach()` or sysfs `echo 1 > /sys/bus/pci/rescan`.
7. DPU boots, services start, reports `dpu_control_plane_state=up`.
8. DPU state transitions to `DPU_READY`.

**DB State Transition:**

| DB Field | Before | During Reboot | After Recovery |
| -------- | :----: | :-----------: | :------------: |
| `dpu_healthy` | `true` | `false` | `true` |
| `reset_status` | `false` | `true` | `true` (cleared by external controller) |
| `PCIE_DETACH_INFO` `dpu_state` | `reattached` | `detaching` → `detached` | `reattached` |

**Error handling:**
- If gNOI service is unreachable: detach PCIe and proceed after timeout.
- If PCIe reattach fails: error handling + restoration mechanism triggered.
- If DPU stuck: hardware watchdog triggers reset (vendor-specific).

---

### Full SmartSwitch Reboot ###

**Description:**
Planned reboot of the entire SmartSwitch (NPU + all DPUs) via CLI: `reboot`. All DPUs are gracefully shut down in parallel before the NPU reboots.

**PMON Sequence:**
1. NPU sends gNOI Reboot RPC (Method: `HALT`) to **all** DPUs in parallel (multiple threads).
2. NPU polls gNOI `RebootStatus` for each DPU until `active=false` and `Status=STATUS_SUCCESS`.
3. Timeout per DPU: `dpu_halt_services_timeout` (default from `platform.json`, typically 300 seconds).
4. For each DPU: PCIe detach via `pci_detach()` or sysfs `echo 1 > /sys/bus/pci/devices/XXXX:XX:XX.X/remove`.
5. NPU proceeds with its own reboot sequence.
6. On NPU boot, PCIe enumeration discovers all DPUs.
7. `chassisd` power-cycles each DPU and performs PCIe reattach.
8. Each DPU boots: midplane attach → SONiC boot → container startup → reports `dpu_control_plane_state=up`.

**DB State Transition:**

| DB Field | Before | During Reboot | After Recovery |
| -------- | :----: | :-----------: | :------------: |
| `dpu_healthy` (all DPUs) | `true` | `false` | `true` (per DPU) |
| `reset_status` (all DPUs) | `false` | `true` | `true` (cleared by external controller per DPU) |
| `PCIE_DETACH_INFO` `dpu_state` (per DPU) | `reattached` | `detaching` → `detached` | `reattached` |

**Error handling:**
- If a DPU does not respond to gNOI Reboot RPC within the timeout: NPU proceeds with PCIe detach and continues the reboot. The unresponsive DPU is cold-booted on NPU recovery.
- If a DPU fails to come back after the full switch reboot: `chassisd` retries power-cycle up to `reset_limit`. If still unresponsive, DPU is marked unrecoverable and an alert is raised (see [Syslog Signatures and Alerting](#syslog-signatures-and-alerting)).
- If the NPU reboot is initiated while a DPU graceful shutdown is in progress: the graceful shutdown completes first, then the NPU reboot proceeds.

---

## Scenario DB State Summary ##

| DPU Scenario | `dpu_control_plane_state` | `dpu_midplane_link_state` | `reset_status` | `dpu_healthy` | PMON Action |
| ------------ | :-----------------------: | :-----------------------: | :------------: | :-----------: | ----------- |
| DPU healthy and running – first boot | up | up | true | true | Set `dpu_healthy=true` after verifying all states |
| External controller programs rules | up | up | false | true | None (external controller clears `reset_status`) |
| DPU crash / unplanned reboot | down | down | true | false | Power-cycle DPU; increment `reset_count` |
| DPU up after crash | up | up | true | true | Set `dpu_healthy=true` after verifying all states |
| DPU stuck (lost connectivity) | down | down | true | false | Power-cycle DPU; increment `reset_count` |
| DPU up after losing connectivity / reboot | up | up | true | true | Set `dpu_healthy=true` after verifying all states |
| DPU control plane restart – critical services | down → up | up | true | false → true | Wait for auto-recovery; set `dpu_healthy=true` on recovery |
| NPU/DPU OS upgrade | down → up | up | true | true | Re-poll DPU states on NPU recovery |
| DPU dead – power cycle | down | down | true | false | Power-cycle DPU; increment `reset_count` |
| DPU dead – unrecoverable | down | down | true | false | `reset_count` reached `reset_limit`; raise alert |
| Full SmartSwitch reboot (planned) | down → up | down → up | true | false → true | gNOI halt; power-cycle; re-verify |

---

## Syslog Signatures and Alerting ##

`chassisd` emits structured syslog messages for DPU failure events. These signatures enable alerting systems (e.g., NetAssist) to create tickets for chronic failures.

| Event | Syslog Severity | Syslog Message Format | Description |
| ----- | :-------------: | --------------------- | ----------- |
| DPU control plane down | WARNING | `chassisd: DPU<index> control plane state changed to down` | DPU control plane detected as down during polling |
| DPU midplane down | WARNING | `chassisd: DPU<index> midplane link state changed to down` | Midplane ping failure detected |
| DPU power-cycle initiated | NOTICE | `chassisd: DPU<index> power-cycle initiated (reset_count=<N>)` | `chassisd` issuing a power-cycle for the DPU |
| DPU recovery successful | NOTICE | `chassisd: DPU<index> recovered successfully (dpu_healthy=true)` | All DPU states verified up after recovery |
| DPU reset limit reached | ALERT | `chassisd: DPU<index> reset limit reached (reset_count=<N>, reset_limit=<L>). DPU marked unrecoverable.` | Chronic failure: DPU has been power-cycled `reset_limit` times without sustained recovery. Requires manual intervention. |
| DPU marked unrecoverable | CRIT | `chassisd: DPU<index> is unrecoverable. Manual intervention required.` | DPU cannot be brought back by automated recovery |
| pmon restart detected | WARNING | `chassisd: pmon restart detected. Re-initializing all DPU states.` | `chassisd` restarted and is re-polling all DPUs |
| PCIe detach detected | WARNING | `chassisd: DPU<index> PCIe detach detected (bus_info=<bus>)` | PCIe link lost for a DPU |
| PCIe reattach successful | NOTICE | `chassisd: DPU<index> PCIe reattach successful` | PCIe link restored after rescan |

**Chronic Failure Alerting:**

When a DPU reaches its `reset_limit`, `chassisd` emits the ALERT-level syslog message above. Alerting systems should key on this signature to:
- Create an incident ticket for the affected DPU
- Flag the switch for hardware investigation
- Notify on-call personnel

The `reset_count` and `reset_limit` values are available in `DPU_RESET_INFO_TABLE` in CHASSIS_STATE_DB for programmatic monitoring.

---

## Platform Behavior: NPU/DPU Reboot Standardization ##

Different SmartSwitch platforms (e.g., Cisco, Nvidia) may have varying default behavior for whether DPUs are rebooted when the NPU reboots. This section documents the **required standard behavior** across all platforms.

### Required Behavior ###

| Trigger | Expected DPU Behavior | Notes |
| ------- | --------------------- | ----- |
| NPU cold reboot (`reboot`) | All DPUs must be gracefully halted (via gNOI) before NPU reboot, then power-cycled on NPU recovery | `chassisd` manages shutdown sequence |
| NPU config reload (`config reload`) | DPUs must **not** be rebooted; only NPU services restart | DPU states are re-polled by `chassisd` after pmon restart |
| NPU kernel crash / panic | DPUs may or may not crash depending on platform hardware coupling | On NPU recovery, `chassisd` must detect DPU states and reboot unresponsive DPUs via gNOI |
| DPU-only reboot (`reboot -d <DPU_ID>`) | Only the targeted DPU is rebooted; other DPUs are unaffected | Standard gNOI halt + power-cycle flow |

### Platform Compliance ###

| Platform | NPU reboot → DPUs halted | NPU config reload → DPUs untouched | NPU crash → DPU reboot on recovery | Status |
| -------- | :----------------------: | :---------------------------------: | :---------------------------------: | ------ |
| Cisco | TBD | TBD | TBD | Pending verification |
| Nvidia | TBD | TBD | TBD | Pending verification |

> **Action Item:** Cross-check and verify the above behaviors on each platform. Update the compliance table with test results.

---

## References ##

- [Smart Switch PMON](../pmon/smartswitch-pmon.md)
- [Smart Switch Graceful Shutdown](../graceful-shutdown/graceful-shutdown.md)
- [Smart Switch Reboot HLD](../reboot/reboot-hld.md)
- [Smart Switch Database Architecture](../smart-switch-database-architecture/smart-switch-database-design.md)
- [Smart Switch IP Address Assignment](../ip-address-assigment/smart-switch-ip-address-assignment.md)
- [Smart Switch DPU Upgrade HLD](../upgrade/dpu-upgrade-hld.md)
