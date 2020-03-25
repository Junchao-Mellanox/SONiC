# SONiC PSU LED Management Design #

### Rev 0.1 ###

### Revision ###

 | Rev |     Date    |       Author       | Change Description                |
 |:---:|:-----------:|:------------------:|-----------------------------------|
 | 0.1 |             |      Chen Junchao  | Initial version                   |


## 1. Overview

The purpose of PSU LED management is to notify user about PSU event by PSU LED or syslog. Current PSU daemon psud need to monitor PSU event (PSU voltage out of range, PSU too hot) and trigger proper actions if necessary.

## 2. PSU event handle

We define a few abnormal PSU events here. When any PSU event happens, syslog should be triggered with "Alert Message", PSU LED should be set to "PSU LED color"; when any PSU restores from previous abnormal state, syslog should be triggered with "Recover Message". PSU LED should be set to green only if there is no any abnormal PSU event happens.

### 2.1 PSU voltage out of range

    Alert Message: PSU voltage warning: <psu_name> voltage out of range, current voltage=<current_voltage>, valid range=[<min_voltage>, <max_voltage>].

    PSU LED color: red.

    Recover Message: PSU voltage warning cleared: <psu_name> voltage is back to normal.

### 2.2 PSU tenmperature too hot

    Alert Message: PSU temperature warning: <psu_name> temperature too hot, temperature=<current_temperature>, threshold=<threshold>.

    PSU LED color: red.

    Recover Message: PSU temperature warning cleared: <psu_name> temperature is back to normal.

### 2.3 Power absence

    Alert Message: Power absence warning: <psu_name> is out of power. 

    PSU LED color: red.

    Recover Message: Power absence warning cleared: <psu_name> power is back to normal.

### 2.4 PSU absence

    Alert Message: PSU absence warning: <psu_name> is not present. 

    PSU LED color: red. (PSU LED might not be available at this point)

    Recover Message: PSU absence warning cleared: <psu_name> is inserted back.

## 3. Platform API change

Some abstract member methods need to be added to [psu_base.py](https://github.com/Azure/sonic-platform-common/blob/master/sonic_platform_base/psu_base.py) and vendor should implement these methods.

```python

class PsuBase(device_base.DeviceBase):
    ...
    def get_temperature(self):
        raise NotImplementedError

    def get_temperature_high_threshold(self):
        raise NotImplementedError

    def get_voltage_high_threshold(self):
        raise NotImplementedError

    def get_voltage_low_threshold(self):
        raise NotImplementedError
    ...

```

## 4. PSU daemon change

We need add a loop to [psud](https://github.com/Azure/sonic-platform-daemons/blob/master/sonic-psud/scripts/psud) to query PSU event every 10 seconds. If any event detects, it should set PSU LED color accordingly and trigger proper syslog.
