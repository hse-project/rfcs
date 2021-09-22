# Introspection APIs

## Problem

HSE does not currently have any way to introspect KVDB/KVS information.

## Requirements

HSE should provide APIs for the following:

- KVDB
  - create parameters
  - runtime parameters
  - home
  - KVS names (done)
- KVS
  - create parameters
  - runtime parameters
  - name

Any REST APIs will support the notion of multiple KVDBs.

## Non-Requirements

- Will not add support for multiple KVDBs in other areas of HSE

## Solution

Create new APIs in C, Python, and REST in order to expose this information. REST
APIs are necessary because you cannot introspect a KVDB open in another process.

### Overview

#### C APIs

The following C APIs will be added behind the experimental wall:

```c
/*
 * Returns a pointer to the kvdb_home member in ikvdb struct.
 */
const char *
hse_kvdb_home_get(const struct hse_kvdb *kvdb);

/*
 * Returns a pointer to the name member in ikvs struct.
 */
const char *
hse_kvs_name_get(const struct hse_kvs *kvs);

/* Returns a JSON representation of the parameter value, ie the value in
 * key=value.
 */
hse_err_t
hse_param_get(const char *param, char *buf, size_t buf_sz, size_t *needed_sz);

/* Returns a JSON representation of the parameter value, ie the value in
 * key=value.
 */
hse_err_t
hse_kvdb_param_get(const struct hse_kvdb *kvdb, const char *param, char *buf, size_t buf_sz, size_t *needed_sz);

/* Returns a JSON representation of the parameter value, ie the value in
 * key=value.
 */
hse_err_t
hse_kvs_param_get(const struct hse_kvs *kvs, const char *param, char *buf, size_t buf_sz, size_t *needed_sz);

typedef const char * hse_mclass_t

#define HSE_MCLASS_CAPACITY_NAME (hse_mclass_t)"capacity"
#define HSE_MCLASS_STAGING_NAME  (hse_mclass_t)"staging"

struct hse_mclass_info {
  uint64_t      mi_allocated_bytes; /**< allocated storage space for a media class */
  uint64_t      mi_used_bytes;      /**< used storage space for a media class */
  uint64_t      mi_reserved[8];     /**< reserved space in struct for future expansion */
  char          mi_path[PATH_MAX];  /**< path to media class */
};

/* Populates the above storage info struct with given information. */
hse_err_t
hse_kvdb_mclass_info_get(const struct hse_kvdb *kvdb, const char *mclass, struct hse_mclass_info *info);
```

#### REST APIs

One issue we have currently that is worth solving in this RFC is that our REST
API assumes there is only one KVDB. We seem to have settled on the idea we want
to support more than one KVDB in the future. Now is the perfect time to retrofit
the REST API.

One issue with name-less KVDBs is that file paths look like parts of a REST
path, which is not good for easy usage. Typically in a well-planned REST route
tree, you would have something like `/kvdb/<specific kvdb>`, where `/kvdb` is
the parent route of all KVDBs. In our current, name-less KVDB situation this
would look like `/kvdb/path/to/kvdb` where `path/to/kvdb` is the KVDB home. This
makes it impossible to parse REST paths correctly. What we can do instead is the
following:

- Add a notion of a REST alias for a KVDB
  - The alias would default to `0` for the first KVDB opened, `1` for the next
    KVDB opened and so on. In the event KVDB 0 is closed and re-opened, KVDB 0
    would become `2`.

The following REST APIs will be added:

- `/params`
  - `GET` - Return a JSON body of all the HSE parameters
- `/params/<param>`
  - `GET` - Return a JSON body of just the parameter that was asked for
- `/kvdb`
  - `GET` - Return a JSON body of all currently opened KVDBs
- `/kvdb/<alias>/home`
  - `GET` - Return a JSON body of the KVDB home
- `/kvdb/<alias>/mclass/<mclass: "capacity" | "staging">/info`
  - `GET` - Return a JSON body of the mclass info struct
- `/kvdb/<alias>/params`
  - `GET` - Return a JSON body of all the KVDB parameters
- `/kvdb/<alias>/params/<param>`
  - `GET` - Return a JSON body of just the parameter that was asked for
  - `PUT` - Change the value of a parameter which is WRITABLE
  - `DELETE` - Set the value of a parameter which is WRITABLE to its default
    value
  - Requirement: create and runtime params must have unique names
- `/kvdb/<alias>/kvs`
  - `GET` - Return a JSON body of all the KVS names in a KVDB
- `/kvdb/<alias>/kvs/<name>/params`
  - `GET` - Return a JSON body of all the KVS parameters for specified KVS
- `/kvdb/<alias>/kvs/<name>/params/<param>`
  - `GET` - Return a JSON body of just the parameter that was asked for
  - `PUT` - Change the value of a parameter which is WRITABLE
  - `DELETE` - Set the value of a parameter which is WRITABLE to its default
    value
  - Requirement: create and runtime params must have unique names

All of the `/params` REST routes will support a query parameter `default`, where
when specified will return the defaults parameters for a type: HSE global
params, KVDB params, or KVS params.

As part of this REST work, I would like to introduce an
[OpenAPI](https://oai.github.io/Documentation/start-here.html) document to the
repo. OpenAPI documents are YAML or JSON files which describe a REST interface.
While we definitely are not ready to publish the REST interface for outside use,
it would definitely help contributors and adventurous users to understand what
REST routes are available and how to use them. The goal would be to document
these proposed APIs, and then as time permits older APIs can be documented as
well.

### Details

The new C-APIs are fairly straightforward and will go from the public interface
to the `ikvdb`/`ikvs` levels. The REST APIs will be wired up upon opening of the
KVDB/KVS.

## Failure and Recovery Handling

N/A

## Testing Guidance

Various API functional tests will be added.
