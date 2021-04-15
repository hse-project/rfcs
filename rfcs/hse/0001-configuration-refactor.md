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

##### API Changes

HSE will lose any notion of `hse_params`. Where `struct hse_params` is being
passed to APIs, that will become a `const char *config`. In order to better
support application developers, HSE will gain two new APIs.

```c
/* hse_config_merge() will allow application developers to pass the config
 * strings that they have configured, and merge a user's given config string
 * with theirs. This function will assume both configs are valid.
 *
 * Ex:
 *
 * char *merged_config;
 * err = hse_config_merge(my_config, user_config, &merged_config);
 * assert(err == 0);
 */
hse_err_t
hse_config_merge(const char *current, const char *incoming, char **merged);

/* hse_config_valid() will take a config string and say whether or not the user
 * has configured a valid string. It returns a boolean. If a non-NULL buf is
 * passed, HSE will fill the buf with useful messages for helping debug the
 * config. Each message will be newline separated until the buf is full. HSE
 * will NUL-terminate the buffer.
 *
 * Ex:
 *
 * char buf[1024];
 * if (!hse_config_valid(my_config, buf, sizeof(buf))) {
 *   fprintf(stderr, "Invalid HSE config string:\n\n%s", buf);
 * }
 */
bool
hse_config_valid(const char *config, char *buf, size_t buf_len);
```

`hse_config_valid()` will be used by the CLI to validate config strings through
a new `hse config validate` subcommand in the future as well.

The same CLI for setting config options of `key=value` will continue to work,
but implemented outside of `libhse`.

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
    "my_kvdb_param": true,
    "kvs": {
      "default": {
        "my_kvs_param": 1
      },
      "kvs1": {
        "my_kvs_open_param": 2
      }
    }
  }
}
```

##### KVDB Home Directory

In the common case where a user wants to keep all of a KVDB's artifacts
together, a `home` has been provided. Per-artifact overrides as described below
will take precedence over `home`. In order to make sure home directories are
exclusive, a PID file will be created called `kvdb.pid`.

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
    "home": "$CWD"
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
        "path": "${kvdb.home}/capacity"
      }
    }
  }
}
```

##### Logging

Outside of KVDBs (after `hse_init()`, before `hse_kvdb_open()`) HSE needs a
place to log messages to. In the past, that has been `stderr`. HSE needs to stop
logging messages to `stderr`/`stdout` by default while also allowing locations
of these messages to be configurable. `hse_init()` will therefore need to take a
config string. `hse_init()` will look at the root-level `logging` key, which
will have the same schema as `kvdb.logging`.

```c
hse_err_t
hse_init(const char *config);
```

HSE should support `logging.path` and `kvdb.logging.path` pointing to the same
file/destination if that is how the user chose to configure their setup. This
will be the default setup.

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
    "path": "$PWD/hse.log in the case of HSE logging and ${kvdb.home}/hse.log in the case of KVDB logging", // both logging.path and kvdb.logging.path will point to the same log file by default
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
      "path": "${kvdb.home}/kvdb.sock"
    }
  }
}
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
