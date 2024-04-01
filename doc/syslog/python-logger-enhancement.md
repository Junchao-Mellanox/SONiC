# SONiC Python logger enhancement #

## Table of Content

### Revision

### Scope

This document describes an enhancement to SONiC Python logger.

### Definitions/Abbreviations

Log identifier: The identifier of a logger instance. Log message printed by the logger instance usually contains the identifier.

### Overview

SONiC provides two Python logger implementations: `sonic_py_common.logger.Logger` and `sonic_py_common.syslogger.SysLogger`. Both of them do not provide the ability to change log level at real time. Sometimes, in order to get more debug information, developer has to manually change the log level in code on a running switch and restart the Python daemon. This is not convenient.

SONiC also provides a C/C++ logger implementation in `sonic-platform-common.common.logger.cpp`. This C/C++ logger implementation is also a wrapper of Linux standard `syslog` which is widely used by swss/syncd. It provides the ability to set log level on fly by starting a thread to listen to CONFIG DB LOGGER table change. SONiC infrastructure also provides the Python wrapper for `sonic-platform-common.common.logger.cpp` which is `swsscommon.Logger`. However, this logger implementation also has some drawbacks:


1. `swsscommon.Logger` assumes redis DB is ready to connect. This is a valid assumption for swss/syncd. But it is not good for a Python logger implementation because some Python script may be called before redis server starting.
2. `swsscommon.Logger` wraps Linux syslog which only support single log identifier for a daemon. 

So, `swsscommon.Logger` is not an option too.

This document describes a Python logger enhancement which allows user setting log level at run time.

### Requirements

- Allow user to change log level of `sonic_py_common.logger.Logger` and `sonic_py_common.syslogger.SysLogger` at run time.
- Logger instance shall use default log level if redis server is not up.

### Architecture Design

Logger/SysLogger class shall start a new thread to listen to redis DB LOGGER table change. The thread shall be shared by all logger instances(both Logger instance and SysLogger instance) within a process. The thread shall be only started on demand.

![python-logger-overview](/doc/syslog/images/python_logger_overview.svg)

### High-Level Design

Here is the flow to create a Logger/SysLogger instance.

![python-logger-create-flow](/doc/syslog/images/python_logger_create_flow.svg)

Here is the flow of Logger config thread.

![python-logger-thread-flow](/doc/syslog/images/python_logger_thread_flow.svg)

Here is an example of how daemon uses the Python logger.

![python-logger-create-example](/doc/syslog/images/python_logger_create_example.svg)

#### Logger/SysLogger class change

New argument shall be added to Logger/SysLogger class constructor.

- enable_config_thread: flag to start config thread. Default to False.

#### Multi processing support

The new implementation supports multi threading well. But it is different in multi processing. In linux, only the current thread will be cloned when creating a sub process. It means that the logger config thread will not be automatically cloned to sub process, logger instance in sub process won't be able to receive the DB change notification.

If user wants to set log level at run time in sub process, some extra steps must be taken:

1. The logger config thread must be reset in sub process context
2. The logger instance must be created in sub process context

Not work:
```python
logger = Logger('test', enable_config_thread=True)

def process_func():
    # Won't start thread because thread is already started in main process
    sub_process_logger = Logger('test', enable_config_thread=True)
    while True:
        # Never print anything because logger will always have level NOTICE 
        # even if user change log level via swssloglevel to INFO
        sub_process_logger.log_info('some message')   

p = multiprocessing.Process(target=process_func)
p.start()
```

Work:
```python
logger = Logger('test', enable_config_thread=True)

def process_func():
    Logger.config_handler.reset() # allow next line to start a new thread
    sub_process_logger = Logger('test', enable_config_thread=True)
    while True:
        # Start to print log if user change log level via swssloglevel to INFO
        sub_process_logger.log_info('some message')   

p = multiprocessing.Process(target=process_func)
p.start()
```

#### Implementation tips

1. All DB related operations are in the config thread. 
2. Config thread shall silently wait until redis is up. Before redis up, logger instance works with its default log level.
3. Config thread shall set daemon=True so that it will automatically exit while daemon exiting.

####

### SAI API

N/A

### Configuration and management

#### Manifest (if the feature is an Application Extension)

N/A

#### CLI/YANG model Enhancements

No CLI change is needed, `swssloglevel` shall be used to change log level.

```
swssloglevel -c <log-identifier> -l <log-level>
```

#### Config DB Enhancements

N/A

### Warmboot and Fastboot Design Impact

N/A

### Memory Consumption

Enable this feature would start a thread for the daemon. The new thread consumes extra but very limited memory.

### Restrictions/Limitations

N/A

### Testing Requirements/Design

#### Unit Test cases

TBD

#### System Test cases

TBD

### Open/Action items - if any
