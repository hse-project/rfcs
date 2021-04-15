# Configuration Refactor

## Problem

HSE 2.0 needs to support configuration strategies that are inline with other
embedded libraries, convenient for application developers, container-friendly,
and open to advanced configuration by end-users.

## Requirements

- HSE should be usable with sane defaults
- Support more than one KVDB per process open at any time in the future without
  breaking current configuration
- Remove configuration files
- Remove notion of workload profiles
- Introduce notion of KVDB home for some use cases
- Allow HSE logs and KVDB logs to go to the same location

## Non-Requirements

- Implement support for multiple KVDBs per process
- Configure `syslog` for users
- Implement support for environment variables of any kind

## Solution

### Overview

HSE's public APIs will have to change in order to support the RFC. APIs that are
added/changing/removed will take into account bindings. Any tests which must be
added/updates will also be updated.

### Details

#### Sane Defaults

Current defaults will stay the same and any path-based configuration like
locations of artifacts will use the current working directory. The location
where a KVDB will store all its artifacts will be referred to as a "KVDB Home".

If a process opens two KVDBs concurrently without changing the KVDB home for
either, the second open should error out appropriately because there is already
a KVDB using the default current working directory as its home.

#### HSE Config at the Application Level

HSE exists to be embedded into applications. Applications need to be the primary
force when it comes to how HSE should be configured while leaving avenues for
end users to have a say. The best way to do this is for application developers
to expose HSE configuration options at the application level. In MongoDB's case,
HSE options that MongoDB wants to expose would belong in the `mongod.conf` file
right next to general MongoDB options. MongoDB will only want to expose stable
configuration options however. For experimental configuration options, we can
use a raw configuration string.

WiredTiger supports a notion of a config string, which it parses into various
configuration values. MongoDB exposes an option to pass a raw WiredTiger
configuration string. MongoDB should in theory do the same thing with HSE.
Advanced end users could then use experimental configuration options to tune HSE
to their liking.

##### `hse_config`

`hse_config` will replace `hse_params` in order to conform to "config string"
nomenclature. `hse_config` will be useful in order to avoid having to re-parse a
config string multiple times. Like config strings, `hse_config` objects should
also be KVDB-scoped. Use one `hse_config` object per KVDB and its KVS.

###### APIs

```c
hse_err_t
hse_config_create(struct hse_config **conf);

hse_err_t
hse_config_from_string(struct hse_config *conf, const char *config_string);

hse_err_t
hse_config_set(struct hse_config *conf, const char *key, const char *value);

bool
hse_config_get(struct hse_config *conf,  char *vbuf, size_t vbuf_sz, size_t *value_len);

bool
hse_config_validate(struct hse_config *conf);
```

These APIs should be thought of as building a config string programmatically.
When using `hse_config_set()`, a user should provide the full key for
configuring an option. For instance, if I want to configure `cn_maint_delay` of
`kvs1` within the KVDB, I would call `hse_config_set()` like so:

```c
hse_config_set(conf, "kvdb.kvs.kvs1.open.cn_maint_delay", "50");
```

The key is what you would use when querying JSON with
[`jq`](https://stedolan.github.io/jq/). This syntax is generally well understood
amongst JSON users.

##### Config String Format

The config string will be a JSON-formatted string. We already have a dependency
on cJSON so this should not be anything new. The config string format should
feel pretty similar to our previous config file format.

All KVDB parameters will be located under the `kvdb` keyword. KVS parameters
will be located under the `kvs` key, which is located under the `kvdb` key. The
`default` keyword under `kvs` will be reserved for parameters which apply to all
KVSs. The outcome of this is that no KVS can be named `default`. KVS-specific
parameters will exist under the name of the KVS within the `kvs` keyword.
KVS-specific parameters will override any listed under `default`.

Config strings will be KVDB-scoped, meaning an application should expose one
config string option per KVDB if it sees fit.

```jsonc
{
  "logging": {
    // later in document
  },
  "kvdb": {
    "logging": {
      // Later in document
    },
    "open": {
      "my_kvdb_open_param": true
    },
    "create": {
      "my_kvdb_create_param": false
    },
    "kvs": {
      "default": {
        "open": {
          "my_kvs_open_param": 1
        },
        "create": {
          "my_kvs_create_params": "hello"
        }
      },
      "kvs1": {
        "open": {
          "my_kvs_open_param": 2
        },
        "create": {
          "my_kvs_create_param": "goodbye"
        }
      }
    }
  }
}
```

##### KVDB Home Directory

In the common case where a user wants to keep all of a KVDB's artifacts
together, a `home` has been provided. Per-artifact overrides as described below
will take precedence over `home`.

Schema:

```jsonc
{
  "kvdb": {
    "home": "string"
  }
}
```

Default Configuration:

```jsonc
{
  "kvdb": {
    "home": "$PWD"
  }
}
```

##### Media Class Storage

Schema:

```jsonc
{
  "kvdb": {
    "storage": {
      "staging": {
        "path": "" // staging won't be configured by default
      },
      "capacity": {
        "path": "string"
      }
    }
  }
}
```

Default Configuration:

```jsonc
{
  "kvdb": {
    "storage": {
      "staging": {
        "path": ""
      },
      "capacity": {
        "path": "$PWD"
      }
    }
  }
}
```

##### Logging

All KVDBs within a process will log to individual files. By default, that
location for each KVDB will be the `$KVDB_HOME/kvdb.log`. The path for the log
file will also be configurable.

Outside of KVDBs (after `hse_init()`, before `hse_kvdb_open()`) HSE needs to a
place to log messages to. In the past, that has been `stderr`. HSE needs to stop
logging messages to `stderr`/`stdout` by default while also allowing locations
of these messages to be configurable. `hse_init()` will therefore need to take
an `hse_config` object. `hse_init()` will look at the root-level `logging` key,
which will have the same schema as `kvdb.logging`, but will use `hse.log` as the
default filename.

```c
hse_err_t
hse_init(struct hse_config *conf);
```

HSE should support `logging.path` and `kvdb.logging.path` pointing to the same
file/destination if that is how the user chose to configure their setup.

Schema:

```jsonc
{
  "logging": {
    "enabled": "boolean",
    "structured": "boolean",
    "destination": "file | syslog | stdout | stderr",
    "path": "when destination == file",
    "level": "[0 - 7]" // support string style as well?
  }
}
```

Default Configuration:

```jsonc
{
  "logging": {
    "enabled": true,
    "structured": false,
    "destination": "file",
    "path": "$PWD/kvdb.log && $PWD/hse.log",
    "level": 7
  }
}
```

Logging can be turned on or off, but can only be _either_ structured or not. HSE
will support 4 destinations for those logs. Log levels will mirror those from
`man syslog`. Setting path when destination is not file will be ignored.

##### UNIX Socket

For each KVDB within a process, there will be a UNIX socket for interacting with
that KVDB.

Schema:

```jsonc
{
  "kvdb": {
    "socket": {
      "path": "string"
    }
  }
}
```

Default Configuration:

```jsonc
{
  "kvdb": {
    "socket": {
      "path": "kvdb.sock"
    }
  }
}
```

#### New Macros for Configuration Keys

In order to better support code completion technologies with the new APIs for
making/opening a KVDB, macros will be defined to string constants for a better
user experience.

```c
// hse/hse.h
#define HSE_CONFIG_LOGGING_ENABLED "logging.enabled"
#define HSE_CONFIG_KVDB_READ_ONLY "read_only"
#define HSE_CONFIG_KVDB_DUR_CAPACITY "dur_capacity"
#define HSE_CONFIG_KVS_CP_FANOUT "cp_fanout"
#define HSE_CONFIG_KVS_CN_MAINT_DELAY "cn_maint_delay"
```

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
