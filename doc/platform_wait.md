# SONiC platform_wait for Nvidia #

## Table of Content

### Revision

### Scope

This document describes the platform_wait function for Nvidia platform.

### Definitions/Abbreviations

N/A

### Overview

`platform_wait` is a script running in PMON container. The purpose of the script is to wait for PMON dependencies to finish their initialization. The script is running at the startup stage of PMON container before any PMON daemon kick in. Currently, PMON daemons only depends on hw-mgmt, so the script `platform_wait` works like this:

1. Get file `/run/hw-management/config/sfp_counter` value as sfp_counter
2. Get file `/run/hw-management/config/module_counter` value as module_counter
3. If module_counter and sfp_counter value are not 0 and module_counter equals to sfp_counter, wait finish; otherwise, repeat step 1~3 until timeout 300 seconds reached

Recently, there are a few new features:

- SDK sysfs feature makes PMON daemons depends on SDK sysfs nodes
- Software control module management feature makes PMON daemons depends on different SDK sysfs nodes based on whether the feature is enabled
- Multi-ASIC feature makes hw-management deprecate file `/run/hw-management/config/module_counter` so that SONiC cannot reply on it anymore

Based on the above, the logic of `platform_wait` must be adjusted to conform with those changes.

### Requirements

1. `platform_wait` shall wait for hw-management related sysfs nodes ready (e.g. PSU related sysfs)
2. `platform_wait` shall wait for SDK related sysfs nodes ready (e.g. Module related sysfs)

### Architecture Design

No architecture change required.

### High-Level Design

`platform_wait` shall be split into 2 stage:

1. wait_hw_management
2. wait_sdk

#### wait_hw_management

When software control module management is disabled, `wait_hw_management` shall do following:

1. Wait for /var/run/hw-management/config/asics_init_done to be value 1

When Software control module management is enabled, minimal driver is not loaded so that asics_init_done and asic_chipup_completed are not available. There is no flag to indicate hw-management readiness in such mode. As SDK is usually started later than hw-management, `platform_wait` shall rely on `wait_sdk` only, and `wait_hw_management` can be ignored in such mode.

#### wait_sdk

SDK team claimed that a SDK sysfs node is ready to use when it is created, so the idea here is to wait the existence of any sysfs nodes that will be used by SONiC.

`wait_sdk` shall do following:

1. Get module number
2. For each module, check sysfs nodes existence. If all sysfs nodes exist, the module is ready.
3. If all modules are ready, all SDK dependencies have been resolved

When Software control module management is enabled, following sysfs nodes shall be checked:

- /sys/module/sx_core/asic0/temperature/input
- /sys/module/sx_core/asic0/module{X}/control
- /sys/module/sx_core/asic0/module{X}/frequency
- /sys/module/sx_core/asic0/module{X}/frequency_support
- /sys/module/sx_core/asic0/module{X}/hw_present
- /sys/module/sx_core/asic0/module{X}/hw_reset
- /sys/module/sx_core/asic0/module{X}/power_good
- /sys/module/sx_core/asic0/module{X}/power_limit
- /sys/module/sx_core/asic0/module{X}/power_on
- /sys/module/sx_core/asic0/module{X}/temperature/input

When Software control module management is disabled, following sysfs nodes shall be checked:

- /sys/module/sx_core/asic0/module{X}/power_mode
- /sys/module/sx_core/asic0/module{X}/power_mode_policy
- /sys/module/sx_core/asic0/module{X}/present
- /sys/module/sx_core/asic0/module{X}/reset
- /sys/module/sx_core/asic0/module{X}/status
- /sys/module/sx_core/asic0/module{X}/statuserror

#### Get module number

Currently, SONiC is using `/run/hw-management/config/module_counter` to get module number. As this file is going to be deprecated, SONiC needs a new way to get module number. `platform.json` can be used to serve this purpose. `platform.json` is a JSON file which contains basic platform information including SFP number.

#### Change platform_wait from shell to python

Currently, `platform_wait` is a shell script which contains following disadvantages:

- Impossible to debug
- Hard to read
- Hard to maintain
- Impossible to cover by unit test

This design shall convert platform_wait to a python script which will address above issues. No change is required for the daemon/script who calls `platform_wait`.

### SAI API

N/A

### Configuration and management

N/A

#### Manifest (if the feature is an Application Extension)

N/A

#### CLI/YANG model Enhancements

N/A

#### Config DB Enhancements

N/A

### Warmboot and Fastboot Design Impact
This design introduces a different mechanism of `platform_wait`, it could affect the start time of PMON. Warmboot and fastboot test cases shall be executed to make sure there is no degradation of control plane/data plane down time.

### Memory Consumption
N/A

### Restrictions/Limitations

### Testing Requirements/Design

Nvidia platform API unit test shall be extended to cover the change.

#### Unit Test cases

#### System Test cases

### Open/Action items - if any

N/A
