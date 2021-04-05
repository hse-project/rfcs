# Configuration Refactor

## Problem

HSE 2.0 will introduce new ways of configuring HSE including a new config file
schema and the introduction of environment variables. In order to support this
new flow, the configuration code needs to be refactored to take these new
requirements into account.

## Requirements

- HSE should be usable with sane defaults
- Support more that one KVDB open at any time in the future without breaking
  current configuration
- Change config file schema to support forward compatibility
- Allow for hierarchical configuration (most important to least important)
  1. Call-site configuration
  2. Environment configuration (Include a way to disable reading environment
     variables entirely)
  3. Config file
  4. Default values
- Remove notion of workload profiles
- Add APIs to introspect HSE configuration options
- Add CLI command to validate configuration file
- Configure logging such that all logs go to a single location
- Configure logging such that it can only be structured or not for the whole
  KVDB

## Non-Requirements

- Implement support multiple KVDBs per process
- Configure `syslog` for users

## Solution

### Overview

HSE's public APIs will have to change in order to support the RFC. APIs that are
added/changing/removed will take into account bindings. Any tests which must be
added/updates will also be updated.

### Details

#### Sane Defaults

Defaults will be taken from current default except in the case of `HSE_ROOT`.
That will be the current working directory.

#### Supporting Multiple KVDBs per Process (Config POV)

In order to be forward compatible with ourselves, we need the config format and
how `HSE_ROOT` is laid out to be constructed in such a way that adding and
supporting multiple open KVDBs per process won't break any APIs, file formats,
or directory structure.

##### `HSE_ROOT`

`HSE_ROOT` is an environment variable which sets the HSE working directory. All
artifacts that HSE creates including logs and data files will be stored here. In
the case of MySQL this would be something like `/var/lib/mysql/`.

```shell
$ tree /var/lib/mysql
/var/lib/mysql
├── hse.conf
├── hse.log
├── hse.sock
└── kvdb1
    ├── staging
    └── capacity
```

With this directory structure, KVDBs have to go back to being named, but this
allows for forward compatibility of multiple KVDBs. KVDBs will share a single
log file, config file, and UNIX socket. This will require us to clean up the
REST routes of the HTTP server to key off of the KVDB name, but given enough
time, this is doable. Log messages will include the KVDB name of which they
apply to, so it should relatively easy to filter messages appropriately.

##### Config File Schema

All KVDB parameters will be located under the `kvdb` keyword and under the name
of the KVDB. KVS parameters will be located under the `kvs` key, which is
located under the `kvdb` key. The `default` keyword under `kvs` will be reserved
for parameters which apply to all KVSs. The outcome of this is that no KVS can
be named `default`. KVS-specific parameters will exist under the name of KVS
within the `kvs` keyword. KVS-specific parameters will override any listed under
`default`.

Example:

```yaml
kvdb:
  kvdb1:
    read_only: true
    kvs:
      kvs1:
        cn_maint_delay: 10
      kvs2:
        cn_node_size_lo: 20
      default:
        cn_main_delay: 5
```

#### Environment Variables

Environment variables are very important for being container friendly and
allowing easy configuration of HSE by end users. They allow for reconfiguration
without recompiling and without editing a configuration file. The naming scheme
of environment variables will be a prefix of `HSE_` and the configuration key
where `.` is changed to `_`. If I was going to set `logging.enabled` through the
environment it would look like `HSE_LOGGING_ENABLED`. For `kvdb.read_only`, it
would be `HSE_KVDB_READ_ONLY`. For setting `kvdb.kvs.cn_maint_delay`, it would
be `HSE_KVDB_KVS_CN_MAINT_DELAY` (`kvs` is under `kvdb` in this new format). For
setting a KVS-specific parameter, wrap the KVS name in 2 underscores on either
side like so, `HSE_KVDB_KVS__KVS1__CN_MAINT_DELAY`. Since we want to be future
forward, the following is equivalent for now,
`HSE_KVDB__KVDB1__KVS__KVS1__CN_MAINT_DELAY`. Notice how the KVDB name has been
wrapped in double underscores similar to the KVS name. The double underscores
should make the variable easy to parse from a visual and a libhse perspective.

#### Logging

##### Schema

```yaml
logging:
  enabled: on | off
  structured: on | off
  destination: file | syslog | stdout | stderr
  path: when destination == file
  level: [0, 7] # support string style as well?
```

##### Default Configuration

```yaml
logging:
  enabled: on
  structured: off
  destination: file
  path: hse.log
  level: 7
```

Logging can be turned on or off, but can only be _either_ structured or not. HSE
will support 4 destinations for those logs. Log levels will mirror those from
`man syslog`. Setting path when destination is not file will be ignored.

A global variable will be added for use internally for logging that is dependent
on being structured. The variable will be set only once at `hse_init()` time.

```c
// hse_util/logging.h
bool logging_is_stuctured = false;
```

#### UNIX Socket

In order to fully support the Linux File System Hierarchy, we need to allow all
artifacts created by HSE to have configurable paths. Earlier we tackled the log
configuration. The UNIX socket will be shared amongst KVDBs within a given
process.

##### Schema

```yaml
socket:
  path: string
```

##### Default Configuration

```yaml
socket:
  path: HSE_ROOT/hse.sock
```

#### New Macros for Configuration Keys

In order to better support code completion technologies with the new APIs for
making/opening a KVDB, macros will be defined to string constants for a better
user experience.

```c
// hse/hse.h
#define HSE_CONF_LOGGING_ENABLED "logging.enabled"
#define HSE_CONF_KVDB_READ_ONLY "read_only"
#define HSE_CONF_KVDB_DUR_CAPACITY "dur_capacity"
#define HSE_CONF_KVS_CP_FANOUT "cp_fanout"
#define HSE_CONF_KVS_CN_MAINT_DELAY "cn_main_delay"
```

#### Initializing HSE

- `hse_init()`

```c
hse_err_t
hse_init(const char *path_to_root, ...) __attribute__(sentinel));
```

Usage:

```c
hse_init("/var/lib/mysql/", HSE_CONF_LOGGING_ENABLED, "true", NULL);
```

- `hse_initv()`[^1]

```c
hse_err_t
hse_initv(const char *path_to_root, size_t nelem, const char *keys, const char *values);
```

Usage:

```c
const char *keys[] = { HSE_CONF_LOGGING_ENABLED };
const char *values[] = { "true" };
assert(sizeof(keys) == sizeof(values));
hse_initv(NULL, N_ELEMS(keys), keys, values);
```

#### Making a KVDB

- `hse_kvdb_make()`

```c
hse_err_t
hse_kvdb_make(const char *name, ...) __attribute__((sentinel));
```

Usage:

```c
hse_kvdb_make("kvdb1", HSE_CONF_KVDB_DUR_CAPACITY, "5", NULL);
```

- `hse_kvdb_makev()`

```c
hse_err_t
hse_kvdb_makev(const char *name, size_t nelem, const char **keys, const char **values) __attribute__((sentinel));
```

Usage:

```c
const char *keys[] = { HSE_CONF_KVDB_DUR_CAPACITY };
const char *values[] = { "5" };
assert(sizeof(keys) == sizeof(values));
hse_kvdb_makev("kvdb1", N_ELEMS(keys), keys, values);
```

#### Making a KVS

- `hse_kvdb_kvs_make()`

Similar to `hse_kvdb_make()`.

- `hse_kvdb_kvs_makev()`

Similar to `hse_kvdb_makev()`.

#### Opening a KVDB

- `hse_kvdb_open()`

Usage:

```cpp
hse_kvdb_open(NULL, HSE_KVDB_READ_ONLY, "true", NULL);
```

- `hse_kvdb_openv()`

Usage:

```c
const char *keys[] = { HSE_KVDB_READ_ONLY };
const char *values[] = { "true" };
assert(sizeof(keys) == sizeof(values));
hse_kvdb_openv(NULL, N_ELEMS(keys), keys, values);
```

#### Opening a KVS

- `hse_kvdb_kvs_open()`

Similar to `hse_kvdb_open()`.

- `hse_kvdb_kvs_openv()`

Similar to `hse_kvdb_openv()`.

---

The new APIs above will deprecate `hse_params`, and it will be removed.

## Failure and Recovery Handling

In the case of a configuration doesn't meet constraints or is incorrect, a
detailed message will be logged with the reasoning and `EINVAL` will be returned
where appropriate.

## Testing Guidance

All tests using any old APIs will be updated to use the new APIs as listed
throughout this document. Testing infrastructure will need to be updated to
account for the new way of configuration.

New tests will be added and old tests will be changed as internals change around
configuration.

## Additional Considerations

- Worth changing make/drop terminology?
  - If we were consistent with databases, the terminology would be create/drop
  - Move to create/destroy? Seem to be using destroy a lot with the discussion
    around libmpool

[^1]:
    The `*v()` APIs will be used by the bindings as variadic arg functions are
    much harder to model.
