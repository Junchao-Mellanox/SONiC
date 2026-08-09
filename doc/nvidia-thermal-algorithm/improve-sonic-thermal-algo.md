# Improve SONiC thermal algorithm design #

## Table of Content 

- [1. Revision](#1-revision)
- [2. Scope](#2-scope)
- [3. Definitions/Abbreviations](#3-definitionsabbreviations)
- [4. Overview](#4-overview)
- [5. Requirements](#5-requirements)
- [6. Architecture Design](#6-architecture-design)
- [7. High-Level Design](#7-high-level-design)
  - [Thermal data source for thermal updater](#thermal-data-source-for-thermal-updater)
  - [Thermal data processing flow for thermal updater](#thermal-data-processing-flow-for-thermal-updater)
    - [Module thermal data processing flow](#module-thermal-data-processing-flow)
    - [ASIC thermal data processing flow](#asic-thermal-data-processing-flow)
  - [thermal-updater change](#thermal-updater-change)
  - [thermal.py change](#thermalpy-change)
  - [hw-management-thermal-updater change](#hw-management-thermal-updater-change)
  - [Nvidia platform files change](#nvidia-platform-files-change)
- [8. SAI API](#8-sai-api)
- [9. Configuration and management](#9-configuration-and-management)
  - [9.1. Manifest](#91-manifest-if-the-feature-is-an-application-extension)
  - [9.2. CLI/YANG model Enhancements](#92-cliyang-model-enhancements)
  - [9.3. Config DB Enhancements](#93-config-db-enhancements)
- [10. Warmboot and Fastboot Design Impact](#10-warmboot-and-fastboot-design-impact)
- [11. Memory Consumption](#11-memory-consumption)
- [12. Restrictions/Limitations](#12-restrictionslimitations)
- [13. Testing Requirements/Design](#13-testing-requirementsdesign)
  - [13.1. Unit Test cases](#131-unit-test-cases)
  - [13.2. System Test cases](#132-system-test-cases)
- [14. Open/Action items](#14-openaction-items---if-any)

### 1. Revision  

### 2. Scope  

This document outlines an enhancement to the SONiC thermal algorithm for the NVIDIA platform. The enhancement centers on ASIC, module temperature data, while other thermal sensors are not included.

### 3. Definitions/Abbreviations 

- Thermal Provider: collecting temperature data from I2C(via SDK driver).
- Thermal Consumer: reading temperature data from places updating by thermal provider.

### 4. Overview 

Currently, multiple processes or tasks are accessing ASIC and module temperature data from real hardware, leading to issues such as:

- Excessive and unnecessary I2C access
- Inconsistent thermal data

The current SONiC thermal algorithm comprises several key components:
- **SDK driver**: Providing sysfs nodes for SONiC to accessing ASIC and module temperature data. 
- **Thermal Provider**
  - **hw-management-thermal-updater**: collecting temperature data for ASIC and firmware-controlled modules, storing to hw-mgmt files.
  - **thermalctld::ThermalUpdater**: collecting temperature data for software-controlled modules and DPUs (smart switch only), storing to hw-mgmt files.
  - **thermalctld::ThermalMonitor**: collecting temperature data from all thermal sensors and stores them in the database, including ASIC and module temperature, storing to redis. CLI `show platform temperature` accesses this data from the redis to present users.
  - **xcvrd::DomInfoUpdateTask**: collecting module temperature data via API `sfp.get_transceiver_dom_real_value`, storing to redis. CLI `show interfaces transceiver eeprom -d` accesses this data from redis to present to users.
  - **xcvrd::DomThermalInfoUpdateTask**: collecting module temperature data via API `sfp.get_temperature`, storing to redis. This task is not enabled on Nvidia platforms.
- **Thermal Consumer**
  - **hw-management-tc**: reading temperature data from hw-mgmt files and performing thermal actions such as adjusting fan speed.
  - **CLI**: reading temperature data from redis and present to users.
  - **SNMP**: reading temperature data from redis and expose via SNMP protocol.

This document outlines improvements to the current SONiC thermal algorithm in the following areas:

- **xcvrd::DomThermalInfoUpdateTask** shall be the only thermal provider for both sw-controlled and fw-controlled module who reads module temp and stores data to redis.
- **thermalctld::ThermalMonitor** shall be the sole ASIC thermal provide who reads ASIC temp and stores data to redis.
- All other thermal providers will either be disabled or become thermal consumer.

This document will focus on Nvidia platform code design.

### 5. Requirements

This section details the requirements for both general SONIC and the Nvidia platform code. Microsoft is responsible for overseeing the general SONiC requirements, while the design pertaining to Microsoft's specific requirements is beyond the scope of this document. The Nvidia-specific requirements are contingent upon changes made to the general SONiC.

1. **xcvrd::DomThermalInfoUpdateTask** shall be enabled and configured with proper polling interval as the only module thermal provider. It handles both software controlled and firmware controlled module. [Nvidia]
2. **thermalctld::ThermalUpdater** shall read module and ASIC temperature from redis [Nvidia]
3. **thermalctld::ThermalUpdater** shall be disabled for liquid cooling system [Nvidia]
4. **hw-management-thermal-updater** shall be disabled [Nvidia]
5. **thermalctld::ThermalMonitor** no longer polling module temperature [Microsoft]
6. **thermalctld::ThermalMonitor** shall support polling different thermal objects at different interval as the only ASIC thermal provider [Microsoft]
7. CLI `show platform temperature` shall retrieve temperature data from the new table provided by #1 due to change in #5, while maintaining its previous output format [Microsoft]
8. SNMP shall retrieve temperature data from the new table provided by #1 due to change in #5 [Microsoft]
9. This feature shall support multi-ASIC [Nvidia] [Microsoft]
10. Dynamic port breakout is not supported

> Note: Due to #5, other platforms also need to enable **xcvrd::DomThermalInfoUpdateTask** so that `show platform temperature` will keep its current output format. Microsoft is responsible for informing all other vendors.

### 6. Architecture Design 

This enhancement does not change existing SONiC architecture. The feature architecture is described in below chart:

![architecture](./arch.svg)

### 7. High-Level Design

#### Thermal data source for thermal updater
Module:

| Thermal data type | Thermal provider | Provider API call | Redis field |
|-------------------|------------------|-------------------|-------------|
| Module temperature | xcvrd::DomThermalInfoUpdateTask | sfp.get_temperature | STATE_DB::TRANSCEIVER_DOM_TEMPERATURE\|Ethernet*.temperature |
| Module temp warning threshold | xcvrd::SfpStateUpdateTask (plug event) | sfp.get_transceiver_threshold_info | STATE_DB::TRANSCEIVER_DOM_THRESHOLD\|Ethernet*.temphighwarning |
| Module temp critical threshold | xcvrd::SfpStateUpdateTask (plug event) | sfp.get_transceiver_threshold_info | STATE_DB::TRANSCEIVER_DOM_THRESHOLD\|Ethernet*.temphighalarm |
| Module vendor name | xcvrd::SfpStateUpdateTask (plug event) | sfp.get_transceiver_info | STATE_DB::TRANSCEIVER_INFO\|Ethernet*.manufacturer |
| Module part number | xcvrd::SfpStateUpdateTask (plug event) | sfp.get_transceiver_info | STATE_DB::TRANSCEIVER_INFO\|Ethernet*.model |

ASIC:

| Thermal data type | Thermal provider | Provider API Call | Redis field|
|-------------------|------------------|-------------------|------------|
| ASIC temperature | thermalctld::ThermalMonitor | thermal.get_temperature | STATE_DB::TEMPERATURE_INFO\|ASIC*.temperature |
| ASIC temp warning threshold | Hard-code | thermal.get_high_threshold | STATE_DB::TEMPERATURE_INFO\|ASIC*.high_threshold |
| ASIC temp critical threshold | Hard-code | thermal.get_high_critical_threshold | STATE_DB::TEMPERATURE_INFO\|ASIC*.critical_high_threshold |



#### Thermal data processing flow for thermal updater

Thermal providers should ensure that data is not fetched from the SDK until the relevant sysfs node is ready. This requirement is upheld by the existing flow, and thus will not be discussed here. This section focuses on the thermal data processing flow for thermal consumers retrieving data from Redis. Thermal providers and consumers operate in separate threads or processes, which can lead to scenarios where the thermal consumer attempts to access redis data before the thermal provider is ready. There are two potential solutions to address this issue:

1. The thermal consumer does not wait for the thermal provider and returns None or N/A if the Redis data is unavailable.
2. Introduce a wait mechanism between the thermal provider and the thermal updater to ensure the readiness of Redis.

| Option | Pros | Cons |
|--------|------|------|
| Option 1 | Simple implementation; No inter-process dependency; Thermal consumer can start immediately | Downstream task needs to handle None/N/A gracefully |
| Option 2 | Guarantees data availability; More reliable data for thermal consumer | Block current thread or process; Adds complexity; Potential startup delay; Requires synchronization mechanism |

Since option 2 introduces more complexities and blocks thread or process, **option 1 is preferred**.

##### Module thermal data processing flow

Before we dive into the flow, we need to address an issue. In Redis, module thermal data is typically indexed by logical port. However, thermal consumers like **thermalctld::ThermalUpdater** manage module temperatures based on physical index. Therefore, there must be a method to map logical ports to physical indexes within the Nvidia platform API. CONFIG_DB::PORT table can be used to build up such mapping.

Module temperature data stored in redis looks like:

```json
{
  "TRANSCEIVER_DOM_TEMPERATURE": {
    "Ethernet0": {
      "temperature": 58.5
    },
    "Ethernet4": {
      "temperature": 50.5
    },
    "Ethernet8": {
      "temperature": 55.5
    }
  }
}
```

CONFIG_DB::PORT table contains data like:

```json
{
  "PORT": {
    "Ethernet0": {
      "index": 1
    },
    "Ethernet4": {
      "index": 1  # Assume there is split
    },
    "Ethernet8": {
      "index": 2
    }
    ...
  }
}
```

Which can be used to build up a python dictionary like (the mapping should be built only once given the dynamic port breakout is not supported):

```
>>> print(port_mapping)
{1: 'Ethernet0', 2: 'Ethernet8' ...}
```

This dictionary makes following code work:

```python
class ThermalUpdater:
    def get_module_temperature(self, module_index):
        logical_port = self.port_mapping[module_index]
        return self.get_module_temperature_from_db(logical_port)
```

Please note that multi-ASIC platforms may have multiple CONFIG_DB instances, the port mapping should be built based on all instances.

The module thermal data flow is described in below chart:

![module-data-flow](./module-data-flow.svg)

> The chart only describes temperature data flow, for threshold, vendor name, the flow is the same.

##### ASIC thermal data processing flow

![asic-data-flow](./asic-data-flow.svg)

#### thermal-updater change

Current behavior:
- It runs when module host management mode is enabled.
- It only collects temperature data for software-controlled modules.
- It collects temperature data from I2C via SDK sysfs nodes

Updated behavior:
- It runs always except liquid cooling systems
- It collects ASIC, modules temperature (both software controlled and firmware controlled).
- It collect temperature data from redis and write to hw-mgmt files

#### thermal.py

Currently, each module has a thermal object defined in `thermal.py` which is called by **thermalctld::ThermalMonitor** . As **thermalctld::ThermalMonitor** no longer collects module temperature, this part is never used in SONiC code. However, sonic-mgmt test case may access this API for test purpose. To avoid test degradation, this API keeps unchanged.

#### hw-management-thermal-updater change
**hw-management-thermal-updater** shall not disabled.

#### Nvidia platform files change

The **hw-management-tc** requires different polling intervals for ASIC and module components. These intervals often differ from the default values in SONiC. Additionally, the same thermal sensor might need varying polling intervals on different platforms. Therefore, it's essential to specify the polling intervals in platform files, as outlined in the `tc_config.json` provided by hw-mgmt for each platform. So, following information shall be provided to each platform files:

1. ASIC polling interval (format to be defined by Microsoft)
2. Module polling interval which potentially enables **xcvrd::DomThermalInfoUpdateTask**

Module polling interval shall be defined in pmon_daemon_control.json as follow:

```json
{
    ...
    "xcvrd": {
        "dom_temperature_poll_interval": 20
    }
    ...
}
```

**thermalctld::ThermalUpdater** shall check this configuration to make sure it is consistent with `tc_config.json`.

### 8. SAI API 

N/A

### 9. Configuration and management 

#### 9.1. Manifest (if the feature is an Application Extension)

N/A

#### 9.2. CLI/YANG model Enhancements 

N/A

#### 9.3. Config DB Enhancements  

N/A
		
### 10. Warmboot and Fastboot Design Impact  

N/A

### Warmboot and Fastboot Performance Impact
No impact to warmboot and fastboot performance.

### 11. Memory Consumption
No extra memory consumption for this feature.

### 12. Restrictions/Limitations  

### 13. Testing Requirements/Design  

#### 13.1. Unit Test cases

All code changes should be covered by new or existing unit test cases.

#### 13.2. System Test cases

Manual test:
1. Verify CLI `show platform temperature`, the output should be the same as before
2. Verify hw-mgmt thermal files are updated with correct value and correct interval
3. Verify that **hw-management-thermal-updater** no longer read ASIC and module temperature
4. Test warm-reboot, fast-reboot, cold reboot, config reload, verify thermal provider and thermal consumer work as expected
5. Test CPU and memory usage for xcvrd and thermalctld, verify the delta is reasonable
6. All test cases shall be performed on SPC1, SPC3, SPC4, SPC5 and smartswitch platforms.
7. All test cases shall be performed with software control and firmware control
8. Verify no new error log introduced

Regression test:
1. Run existing platform related tests, verify no degradation

### 14. Open/Action items - if any 

