# Thermal updater for Software control module management mode #

## Table of Content

### Revision

### Scope

This document describes the thermal updater for Software control module management mode. This design is for Nvidia platform only.

### Definitions/Abbreviations

N/A

### Overview

hw-management-tc is the service which runs thermal control algorithm in sonic. ASIC and SFP module temperature/threshold are considered as input parameter for thermal control algorithm. Currently, driver is responsible for providing ASIC and SFP module temperature/threshold via sysfs. However, driver cannot provide such information under Software control module management mode. This document describes a new task thermal updater which handles ASIC and SFP module temperature/threshold update for hw-management-tc.

### Requirements

1. Thermal updater task shall update ASIC and SFP module temperature/threshold periodically
2. Thermal updater task shall create ASIC and SFP module temperature/threshold data on startup and initialize with value 0
3. Thermal updater task shall set ASIC and SFP module temperature/threshold data to value 0 when module is plugged out
4. Thermal updater task shall suspend hw-management-tc on graceful exit (hw-management-tc set fan speed to 100% on suspend)
5. Thermal updater task shall be auto restarted on crash

### Architecture Design

The current architecture is not changed

### High-Level Design

Thermal updater is a thread running inside thermalctld. The code of thermal updater is under Nvidia platform API, no code change for other component. Thermal updater shall be created/destroyed by existing platform API:

- `thermal_manager_base.ThermalManagerBase.initialize`: start thermal updater
- `thermal_manager_base.ThermalManagerBase.initialize`: destroy thermal updater

Thermal updater shall be started only if software control module management mode is enabled.

#### Thermal updater task implementation

Thermal updater task flow:

![thermal-updater](/doc/thermal_updater/thermal_updater.svg).

##### Init/clean thermal data

Thermal updater task shall initialize thermal data for ASIC and SFP module.

hw-management has provided following interfaces to update thermal data.

```
module_data_set_module_counter  # configure module counter
thermal_data_set_asic           # set ASIC thermal data
thermal_data_set_module         # set module thermal data
thermal_data_clean_asic         # clean ASIC thermal data
thermal_data_clean_module       # clean module thermal data
```

- Init thermal data for ASIC

```
thermal_data_clean_asic(args...)
```

- Init thermal data for each module

```
thermal_data_clean_module(args...)
```

##### Wait sonic init done

Under software control module management mode, sonic is responsible for initialize modules. Thermal updater task has to wait module EEPROM ready.

##### Update ASIC thermal data

In the main loop, thermal updater task shall check ASIC temperature information via SDK sysfs and update hw-management-tc periodically.

SDK sysfs:

- temperature: /sys/module/sx_core/asic${id}/temperature/input
- emergency threshold: /sys/module/sx_core/asic${id}/temperature/emergency (Use 105C if sysfs does not exist)
- critical threshold: /sys/module/sx_core/asic${id}/temperature/crit (Use 120C if sysfs does not exist)

##### Update Module thermal data

In the main loop, thermal updater task shall get each module temperature information and update hw-management-tc periodically.

For module under firmware control, thermal data shall get from SDK sysfs:

- temperature: /sys/module/sx_core/asic${id}/module${id}/temperature/input
- emergency threshold: /sys/module/sx_core/asic${id}/module${id}/temperature/emergency (Use 70C if sysfs does not exist)
- critical threshold: /sys/module/sx_core/asic${id}/module${id}/temperature/crit (Use 80C if sysfs does not exist)

For module under software control, thermal data shall get from module EEPROM. In current sonic-platform-common, it already provides implementation of getting temperature and threshold.

### SAI API

N/A

### Configuration and management

#### Manifest (if the feature is an Application Extension)

N/A

#### CLI/YANG model Enhancements

N/A

#### Config DB Enhancements

N/A

### Warmboot and Fastboot Design Impact

N/A

### Memory Consumption

No memory consumption is expected when the feature is disabled via compilation and no growing memory consumption while feature is disabled by configuration.

### Restrictions/Limitations

N/A

### Testing Requirements/Design

#### Unit Test cases

TBD

#### System Test cases

### Open/Action items - if any
