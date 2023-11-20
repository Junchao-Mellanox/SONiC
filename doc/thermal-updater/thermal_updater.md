# Thermal updater for independent module #

## Table of Content

### Revision

### Scope

This document describes the thermal updater for independent module. This design is for Nvidia platform only.

### Definitions/Abbreviations

N/A

### Overview

hw-management-tc is the service which runs thermal control algorithm in sonic. ASIC and SFP module temperature/threshold are considered as input parameter for thermal control algorithm. Currently, driver is responsible for providing ASIC and SFP module temperature/threshold via sysfs. However, driver cannot provide such information under CMIS host management enabled platform. This document describes a new task thermal updater which handles ASIC and SFP module temperature/threshold update for hw-management-tc.

### Requirements

1. Thermal updater task shall update ASIC and SFP module temperature/threshold periodically
2. Thermal updater task shall create ASIC and SFP module temperature/threshold thermal data on startup and initialize with value 0
3. Thermal updater task shall set ASIC and SFP module temperature/threshold thermal data to value 0 when module is plugged out
4. Thermal updater task shall suspend hw-management-tc on graceful exit (hw-management-tc set fan speed to 100% on suspend)
5. Thermal updater task shall be auto restarted on crash
6. Thermal updater task shall work for CMIS host management enabled platform

### Architecture Design

The current SONIC architecture is not changed

### High-Level Design

#### Thermal updater task implementation

Thermal updater task flow:

![thermal-updater](/doc/thermal-updater/thermal_updater.svg).

##### Init thermal data

Thermal updater task shall initialize thermal data for ASIC and SFP modules.

hw-management shall provide an interface to update thermal data. The interface looks like this:

```
- hw-management-independent-mode-set(index, type, value)   - this API will create (if does not exist) or modify the relevant file in /var/run/hw-management/thermal/
  where:
  - index (0 – ASIC, 1..n – modules)
  - type – sting ‘crit|emergency|input’
  - value - value to set (if there is a problem to provide input in Celsius, it can be scaled inside API).

Note: if module is not inserted – OS process should inject zeros for all related attributes.

- hw-management-independent-mode-remove(index, type) – to remove file (likely it’ll not some individual operation fro removing specific file, but rather it will work for all files)
  where:
  - index  (0 – ASIC, 1..n – modules)
  - type – sting ‘crit|emergency|fault|input’.
```

- Init thermal data for ASIC

```
hw-management-independent-mode-set(0, input, 0)
hw-management-independent-mode-set(0, emergency, 0)
hw-management-independent-mode-set(0, crit, 0)
```

- Init thermal data for each module

```
hw-management-independent-mode-set(<module_index>, input, 0)
hw-management-independent-mode-set(<module_index>, emergency, 0)
hw-management-independent-mode-set(<module_index>, crit, 0)
```

##### Check mode (CMIS host management enabled or disabled)

Check SAI profile to understand the current mode.

> Note: This is the global flag. While CMIS host management is enabled, all CMIS modules shall be managed by NOS; other types of cables shall still be managed by firmware.

##### Wait sonic init done

Under CMIS host management mode, sonic is responsible for initialize modules. Thermal updater task has to wait module initialization done.

##### Update ASIC thermal data

In the main loop, thermal updater task shall check ASIC temperature information via SDK sysfs and update hw-management-tc via `hw-management-independent-mode-set` periodically.

SDK sysfs:

- temperature: /sys/module/sx_core/asic${id}/temperature/input
- emergency threshold: /sys/module/sx_core/asic${id}/temperature/emergency (Use 105C if SDK does not provide)
- critical threshold: /sys/module/sx_core/asic${id}/temperature/crit (Use 120C if SDK does not provide)

If thermal updater fails to get ASIC thermal data, it shall send the fault status to hw-management-tc via `hw-management-independent-mode-set`.

##### Update Module thermal data

In the main loop, thermal updater task shall get each module temperature information and update hw-management-tc via `hw-management-independent-mode-set` periodically.

For module under firmware management, thermal data shall get from SDK sysfs:

- temperature: /sys/module/sx_core/asic${id}/module${id}/temperature/input
- emergency threshold: /sys/module/sx_core/asic${id}/module${id}/temperature/emergency (Use 70C if SDK does not provide)
- critical threshold: /sys/module/sx_core/asic${id}/module${id}/temperature/crit (Use 80C if SDK does not provide)

For module under software management(currently only CMIS), thermal data shall get from module EEPROM. In current sonic-platform-common, it already provides implementation of get temperature and threshold.

If thermal updater fails to get module thermal data, it shall send the fault status to hw-management-tc via `hw-management-independent-mode-set`.

#### Deploy thermal updater task

Thermal updater task shall be deployed to thermalctld running in PMON container. Thermal update task shall be controlled by following platform API:

- `thermal_manager_base.initialize`: start thermal updater task.
- `thermal_manager_base.deinitialize`: graceful stop thermal updater task.

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
