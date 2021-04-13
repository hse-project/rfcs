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
- Allow configuration only at call-site with opportunity for extension that can
  be exposed by application developers to end users

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
locations of artifacts will use the current working directory. Within the
current working directory, each KVDB within a process will produce a directory
for all of its artifacts, and it will be named with the name of the KVDB. The
location where a KVDB will store all its artifacts will be referred to as a
"KVDB Home".

```shell
$PWD/kvdb1
```

#### HSE Config at the Application Level

HSE exists to be embedded into applications. Applications need to be the primary
force when it comes to how HSE should be configured while leaving avenues for
end users to have a say. The best way to do this is for application developers
to expose HSE configuration options at the application level. In MongoDB's case,
HSE options that MongoDB wants to expose would belong in the `mongodb.conf` file
right next to general MongoDB options. MongoDB will only want to expose stable
configuration options however. For experimental configuration options, we can
use a raw configuration string.

WiredTiger supports a notion of a config string, which it parses into various
configuration values. MongoDB exposes an option to pass a raw WiredTiger
configuration string. MongoDB should in theory do the same thing with HSE.
Advanced end users could then use experimental configuration options to tune HSE
to their liking.

##### Config String Format

The config string will be a JSON-formatted string. We already have a dependency
on cJSON so this should not be anything new. The config string format should
feel pretty similar to our previous config file format.

All KVDB parameters will be located under the `kvdb` keyword and under the name
of the KVDB. KVS parameters will be located under the `kvs` key, which is
located under the `kvdb` key. The `default` keyword under `kvs` will be reserved
for parameters which apply to all KVSs. The outcome of this is that no KVS can
be named `default`. KVS-specific parameters will exist under the name of KVS
within the `kvs` keyword. KVS-specific parameters will override any listed under
`default`.

```jsonc
{
  "logging": {
    // Later in document
  },
  "kvdb": {
    "kvdb1": {
      "my_kvdb_param": true,
      "kvs": {
        "default": {
          "my_kvs_param": 1
        },
        "kvs1": {
          "my_kvs_param": 1
        }
      }
    }
  }
}
```

##### KVDB Home Directory

In the case, a user wants to keep all of a KVDB's artifacts together, a `home`
has been provided. Per-artifact overrides as described below will take
precedence over `home`.

Schema:

```jsonc
{
  "kvdb": {
    "kvdb1": {
      "home": "string"
    }
  }
}
```

Default Configuration:

```jsonc
{
  "kvdb": {
    "kvdb1": {
      "home": "$PWD/kvdb1"
    }
  }
}
```

##### Media Class Storage

Schema:

```jsonc
{
  "kvdb": {
    "kvdb1": {
      "storage": {
        "staging": {
          "path": "string"
        },
        "capacity": {
          "path": "string"
        }
      }
    }
  }
}
```

Default Configuration:

```jsonc
{
  "kvdb": {
    "kvdb1": {
      "storage": {
        "staging": {
          "path": "$PWD/kvdb1/staging"
        },
        "capacity": {
          "path": "$PWD/kvdb1/capacity"
        }
      }
    }
  }
}
```

##### Logging

All KVDBs within a process will share a single log location. By default, that
location will be the current working directory in a file called `hse.log`. The
path for the log file will also be configurable. All log messages will be
accompanied by the name of the KVDB that they pertain to as to make it easy to
grep for messages pertaining to each KVDB.

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
    "path": "hse.log",
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
    "kvdb1": {
      "socket": {
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
    "kvdb1": {
      "socket": {
        "path": "hse.sock"
      }
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

#### `hse_config`

`hse_config` will replace `hse_params` in order to conform to "config string"
nomenclature. `hse_config` will be useful in order to avoid having to re-parse a
config string multiple times.

##### APIs

```c
hse_err_t
hse_config_create(struct hse_config **conf);

hse_err_t
hse_config_from_string(struct hse_config *conf, const char *config_string);

hse_err_t
hse_config_set(struct hse_config *conf, const char *key, const char *value);

hse_err_t
hse_config_get(struct hse_config *conf, const char *key);
```

These APIs should be thought of as building a config string programmatically.
When using `hse_config_set()`, a user should provide the full key for
configuring an option. For instance, if I want to configure `cn_maint_delay` of
`kvs1` within `kvdb1`, I would call `hse_config_set()` like so:

```c
hse_config_set(conf, "kvdb.kvdb1.kvs.kvs1.cn_maint_delay", "50");
```

The key is what you would use when querying JSON with
[`jq`](https://stedolan.github.io/jq/). This syntax is generally well understood
amongst JSON users.

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
- Leaving room for a potential `HSE_CONFIG` environment variable in the future
  whose value would be a JSON string.
- Logs on a per-kvdb basis instead of an entire subsystem basis?
- Could we make it so that `hse_config`-related code was not initialized during
  `hse_init()`, but was instead purely static? `hse_init()` would then take an
  `hse_config` object. If we could pull that off, logging could begin much
  earlier in the HSE lifecycle.
- Tom made a mention of RockDB's logging API:
  https://github.com/facebook/rocksdb/wiki/Logger
  - Is it worth doing something similar?
