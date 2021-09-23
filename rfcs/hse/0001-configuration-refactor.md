# Configuration Refactor

## Problem

HSE 2.0 needs to support configuration strategies that are inline with other
embedded libraries, convenient for application developers, container-friendly,
and open to advanced configuration by end-users.

## Requirements

- HSE should be usable with sane defaults
- Support more than one KVDB per process open at any time in the future without
  breaking current configuration
- Remove notion of workload profiles
- Introduce notion of KVDB home for some use cases

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
passed to APIs, that will become a `const char *config`. HSE APIs will change in
the following ways:

```c
hse_err_t
hse_init(const char *config, size_t paramc, const char *const *paramv);

hse_err_t
hse_kvdb_create(const char *kvdb_home, size_t paramc, const char *const *paramv);

hse_err_t
hse_kvdb_open(const char *kvdb_home, size_t paramc, const char *const *paramv, struct hse_kvdb **kvdb);

hse_err_t
hse_kvdb_kvs_create(struct hse_kvdb *kvdb, const char *kvs_name, size_t paramc, const char *const *paramv);

hse_err_t
hse_kvdb_kvs_open(struct hse_kvdb *kvdb, const char *kvs_name, size_t paramc, const char *const *paramv, struct hse_kvs **kvs);
```

`paramv` will be an array of `key=value` strings where `paramc` is the size of
said array. The key will be something like `prefix.length` and the value will be
something like `5`. Combined that will look like `prefix.length=5`.

##### Config File Formats

HSE will have two config file formats. One for HSE global parameters and another
for KVDB/KVS parameters. They will both be JSON files.

###### hse.conf

```jsonc
{
  "logging": {
    // later in document
  }
  // ...
}
```

###### kvdb.conf

```jsonc
{
  "read_only": "...",
  "kvs": {
    "default": {
      "prefix": {
        "length": 5
      }
    },
    "mykvs": {
      "prefix": {
        "length": 3
      }
    }
  }
}
```

KVS parameters will be located under the `kvs` key, which is located under the
`kvdb` key. The `default` keyword under `kvs` will be reserved for parameters
which apply to all KVSs. The outcome of this is that no KVS can be named
`default`. KVS-specific parameters will exist under the name of the KVS within
the `kvs` keyword. KVS-specific parameters will override any listed under
`default`.

##### KVDB Home Directory

In the common case where a user wants to keep all of a KVDB's artifacts
together, a `home` has been provided. Per-artifact overrides as described below
will take precedence over `home`. In order to make sure home directories are
exclusive, a PID file will be created called `kvdb.pid`.

##### Media Class Storage

Schema:

```jsonc
{
  "storage": {
    "staging": {
      "path": "string" // staging won't be configured by default
    },
    "capacity": {
      "path": "string"
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
        "path": null
      },
      "capacity": {
        "path": "${kvdb_home}/capacity"
      }
    }
  }
}
```

##### Logging

Logs by default will continue to go to `syslog`. Logging will be configured in
the `hse.conf` file under the `logging` key or `logging.xxxxx=yyyyy` in the
`hse_init()` API.

Schema:

```jsonc
{
  "logging": {
    "enabled": "boolean",
    "structured": "boolean",
    "destination": "file | syslog | stdout | stderr",
    "path": "when destination == file",
    "level": "[0 - 7]"
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
    "path": "${kvdb_home}", // both logging.path and kvdb.logging.path will point to the same log file by default
    "level": 7
  }
}
```

Logging can be turned on or off, but can only be _either_ structured or not. HSE
will support 4 destinations for those logs. Log levels will mirror those from
`man syslog`. Setting path when destination is not file will be ignored.

##### UNIX Socket

Unix socket paths are limited to 107 characters plus 1 for the `NUL` byte. By
default socket paths will be `/tmp/hse-$PID.sock`, but will be configurable to a
custom path as long as said path is within the length limits. The socket can be
disabled using the `socket.enabled` parameter.

Schema:

```jsonc
{
  "socket": {
    "enabled": "boolean",
    "path": "string"
  }
}
```

Default Configuration:

```jsonc
{
  "socket": {
    "enabled": true,
    "path": "/tmp/hse-$PID.sock"
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
