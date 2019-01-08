# lua-resty-mlcache

[![Build Status][badge-travis-image]][badge-travis-url]

Fast and automated layered caching for OpenResty.

This library can be manipulated as a key/value store caching scalar Lua types
and tables, combining the power of the [lua_shared_dict] API and
[lua-resty-lrucache], which results in an extremely performant and flexible
caching solution.

Features:

- Caching and negative caching with TTLs.
- Built-in mutex via [lua-resty-lock] to prevent dog-pile effects to your
  database/backend on cache misses.
- Built-in inter-workers communication to propagate cache invalidations
  and allow workers to update their L1 (lua-resty-lrucache) caches upon changes
  (`set()`, `delete()`).
- Support for split hits and misses caching queues.
- Multiple isolated instances can be created to hold various types of data
  while relying on the *same* `lua_shared_dict` L2 cache.

Illustration of the various caching levels built into this library:

```
┌─────────────────────────────────────────────────┐
│ Nginx                                           │
│       ┌───────────┐ ┌───────────┐ ┌───────────┐ │
│       │worker     │ │worker     │ │worker     │ │
│ L1    │           │ │           │ │           │ │
│       │ Lua cache │ │ Lua cache │ │ Lua cache │ │
│       └───────────┘ └───────────┘ └───────────┘ │
│             │             │             │       │
│             ▼             ▼             ▼       │
│       ┌───────────────────────────────────────┐ │
│       │                                       │ │
│ L2    │           lua_shared_dict             │ │
│       │                                       │ │
│       └───────────────────────────────────────┘ │
│                           │ mutex               │
│                           ▼                     │
│                  ┌──────────────────┐           │
│                  │     callback     │           │
│                  └────────┬─────────┘           │
└───────────────────────────┼─────────────────────┘
                            │
  L3                        │   I/O fetch
                            ▼

                   Database, API, DNS, Disk, any I/O...
```

The cache level hierarchy is:
- **L1**: Least-Recently-Used Lua VM cache using [lua-resty-lrucache].
   Provides the fastest lookup if populated, and avoids exhausting the workers'
   Lua VM memory.
- **L2**: `lua_shared_dict` memory zone shared by all workers. This level
   is only accessed if L1 was a miss, and prevents workers from requesting the
   L3 cache.
- **L3**: a custom function that will only be run by a single worker
   to avoid the dog-pile effect on your database/backend
   (via [lua-resty-lock]). Values fetched via L3 will be set to the L2 cache
   for other workers to retrieve.

# Table of Contents

- [Synopsis](#synopsis)
- [Requirements](#requirements)
- [Installation](#installation)
- [Methods](#methods)
    - [new](#new)
    - [get](#get)
    - [peek](#peek)
    - [set](#set)
    - [delete](#delete)
    - [purge](#purge)
    - [update](#update)
- [Changelog](#changelog)
- [License](#license)

# Synopsis

```
# nginx.conf

http {
    # you do not need to configure the following line when you
    # use LuaRocks or opm.
    lua_package_path "/path/to/lua-resty-mlcache/lib/?.lua;;";

    # 'on' already is the default for this directive. If 'off', the L1 cache
    # will be inefective since the Lua VM will be re-created for every
    # request. This is fine during development, but ensure production is 'on'.
    lua_code_cache on;

    lua_shared_dict cache_dict 1m;

    init_by_lua_block {
        local mlcache = require "resty.mlcache"

        local cache, err = mlcache.new("my_cache", "cache_dict", {
            lru_size = 500,    -- size of the L1 (Lua VM) cache
            ttl      = 3600,   -- 1h ttl for hits
            neg_ttl  = 30,     -- 30s ttl for misses
        })
        if err then

        end

        -- we put our instance in the global table for brivety in
        -- this example, but prefer an upvalue to one of your modules
        -- as recommended by ngx_lua
        _G.cache = cache
    }

    server {
        listen 8080;

        location / {
            content_by_lua_block {
                local function callback(username)
                    -- this only runs *once* until the key expires, so
                    -- do expensive operations like connecting to a remote
                    -- backend here. i.e: call a MySQL server in this callback
                    return db:get_user(username) -- { name = "John Doe", email = "john@example.com" }
                end

                -- this call will try L1 and L2 before running the callback (L3)
                -- the returned value will then be stored in L2 and L1
                -- for the next request.
                local user, err = cache:get("my_key", nil, callback, "John Doe")

                ngx.say(user.username) -- "John Doe"
            }
        }
    }
}
```

[Back to TOC](#table-of-contents)

# Requirements

* OpenResty >= `1.11.2.2`
    * ngx_lua
    * lua-resty-lrucache
    * lua-resty-lock

Tests matrix results:

| OpenResty   | Compatibility
|------------:|:--------------------|
| <           | not tested
| `1.11.2.2`  | :heavy_check_mark:
| `1.11.2.3`  | :heavy_check_mark:
| `1.11.2.4`  | :heavy_check_mark:
| `1.11.2.5`  | :heavy_check_mark:
| `1.13.6.1`  | :heavy_check_mark:
| `1.13.6.2`  | :heavy_check_mark:
| >           | not tested

[Back to TOC](#table-of-contents)

# Installation

With [LuaRocks](https://luarocks.org/):

```
$ luarocks install lua-resty-mlcache
```

Or via [opm](https://github.com/openresty/opm):

```
$ opm get thibaultcha/lua-resty-mlcache
```

Or manually:

Once you have a local copy of this module's `lib/` directory, add it to your
`LUA_PATH` (or `lua_package_path` directive for OpenResty):

```
/path/to/lib/?.lua;
```

[Back to TOC](#table-of-contents)

# Methods

new
---
**syntax:** `cache, err = mlcache.new(name, shm, opts?)`

Create a new mlcache instance. If failed, returns `nil` and a string
describing the error.

The first argument `name` is an arbitrary name of your choosing for this cache,
and must be a string. Each mlcache instance namespaces the values it holds
according to its name, so several instances with the same name will
share the same data.

The second argument `shm` is the name of the `lua_shared_dict` shared memory
zone. Several instances of mlcache can use the same shm (values will be
namespaced).

The third argument `opts` is optional. If provided, it must be a table
holding the desired options for this instance. The possible options are:

- `lru_size`: a number defining the size of the underlying L1 cache
  (lua-resty-lrucache instance). This size is the maximal number of items
  that the L1 cache can hold.
  **Default:** `100`.
- `ttl`: a number specifying the expiration time period of the cached
  values. The unit is seconds, but accepts fractional number parts, like
  `0.3`. A `ttl` of `0` means the cached values will never expire.
  **Default:** `30`.
- `neg_ttl`: a number specifying the expiration time period of the cached
  misses (when the L3 callback returns `nil`). The unit is seconds, but
  accepts fractional number parts, like `0.3`. A `neg_ttl` of `0` means the
  cached misses will never expire.
  **Default:** `5`.
- `resurrect_ttl`: _optional_ number. When specified, the mlcache instance will
  attempt to resurrect stale values when the L3 callback returns `nil, err`
  (soft errors). More details are available for this option in the
  [get()](#get) section. The unit is seconds, but accepts fractional number
  parts, like `0.3`.
- `lru`: _optional_. A lua-resty-lrucache instance of your choosing. If
  specified, mlcache will not instantiate an LRU. One can use this value to use
  the `resty.lrucache.pureffi` implementation of lua-resty-lrucache if desired.
- `shm_set_tries`: the number of tries for the lua_shared_dict `set()`
  operation. When the lua_shared_dict is full, it attempts to free up to 30
  items from its queue. When the value being set is much larger than the freed
  space, this option allows mlcache to retry the operation (and free more slots)
  until the maximum number of tries is reached or enough memory was freed for
  the value to fit.
  **Default**: `3`.
- `shm_miss`: _optional_ string. The name of a `lua_shared_dict`. When
  specified, misses (callbacks returning `nil`) will be cached in this separate
  lua_shared_dict. This is useful to ensure that a large number of cache misses
  (e.g. triggered by malicions clients) does not evict too many cached items
  (hits) from the lua_shared_dict specified in `shm`.
- `shm_locks`: _optional_ string. The name of a `lua_shared_dict`. When
  specified, lua-resty-lock will use this shared dict to store its locks. This
  option can help reducing cache churning: when the L2 cache (shm) is full,
  every insertion (such as locks created by concurrent accesses triggering L3
  callbacks) purges the oldest 30 accessed items. These purged items are most
  likely to be previously (and valuable) cached values. By isolating locks in a
  separate shared dict, workloads experiencing cache churning can mitigate this
  effect.
- `resty_lock_opts`: _optional_ table. Options for [lua-resty-lock] instances.
  When mlcache runs the L3 callback, it uses lua-resty-lock to ensure that a
  single worker runs the provided callback.
- `ipc_shm`: _optional_ string. If you wish to use [set()](#set),
  [delete()](#delete), or [purge()](#purge), you must provide an IPC
  (Inter-Process Communication) mechanism for workers to synchronize and
  invalidate their L1 caches. This module bundles an "off-the-shelf" IPC
  library, and you can enable it by specifying a dedicated `lua_shared_dict` in
  this option. Several mlcache instances can use the same shared dict (events
  will be namespaced), but no other actor than mlcache should tamper with it.
- `ipc`: _optional_ table. Like the above `ipc_shm` option, but lets you use
  the IPC library of your choice to propagate inter-worker events.
- `l1_serializer`: _optional_ function. Its signature and accepted values are
  documented under the [get()](#get) method, along with an example. If
  specified, this function will be called each time a value is promoted from the
  L2 cache into the L1 (worker Lua VM). This function can perform arbitrary
  de-serialization of the cached item to transform it into any Lua object
  _before_ storing it into the L1 cache. It can thus avoid your application from
  having to repeat such transformations on every request, such as creating
  tables, cdata objects, loading new Lua code, etc...

Example:

```lua
local mlcache = require "resty.mlcache"

local cache, err = mlcache.new("my_cache", "cache_shared_dict", {
    lru_size = 1000, -- hold up to 1000 items in the L1 cache (Lua VM)
    ttl      = 3600, -- caches scalar types and tables for 1h
    neg_ttl  = 60    -- caches nil values for 60s
})
if not cache then
    error("could not create mlcache: " .. err)
end
```

You can create several mlcache instances relying on the same underlying
`lua_shared_dict` shared memory zone:

```lua
local mlcache = require "mlcache"

local cache_1 = mlcache.new("cache_1", "cache_shared_dict", { lru_size = 100 })
local cache_2 = mlcache.new("cache_2", "cache_shared_dict", { lru_size = 1e5 })
```

In the above example, `cache_1` is ideal for holding a few, very large values.
`cache_2` can be used to hold a large number of small values. Both instances
will rely on the same shm: `lua_shared_dict cache_shared_dict 2048m;`. Even if
you use identical keys in both caches, they will not conflict with each other
since they each have a different namespace.

This other example instantiates an mlcache using the bundled IPC module for
inter-workers invalidation events (so we can use [set()](#set),
[delete()](#delete), and [purge()](#purge)):

```lua
local mlcache = require "resty.mlcache"

local cache, err = mlcache.new("my_cache_with_ipc", "cache_shared_dict", {
    lru_size = 1000,
    ipc_shm = "ipc_shared_dict"
})
```

**Note:** for the L1 cache to be effective, ensure that
[lua_code_cache](https://github.com/openresty/lua-nginx-module#lua_code_cache)
is enabled (which is the default). If you turn off this directive during
development, mlcache will work, but L1 caching will be ineffective since a new
Lua VM will be created for every request.

[Back to TOC](#table-of-contents)

get
---
**syntax:** `value, err, hit_level = cache:get(key, opts?, callback, ...)`

Perform a cache lookup. This is the primary and most efficient method of this
module. A typical pattern is to *not* call [set()](#set), and let [get()](#get)
perform all the work.

When it succeeds, it returns `value` and no error. **Because `nil` values from
the L3 callback are cached to signify misses, `value` can be nil, hence one
must rely on the second return value `err` to determine if this method
succeeded or not**.

The third return value is a number which is set if no error was encountered.
It indicated the level at which the value was fetched: `1` for L1, `2` for L2,
and `3` for L3.

If an error is encountered, this method returns `nil` plus a string describing
the error.

The first argument `key` is a string. Each value must be stored under a unique
key.

The second argument `opts` is optional. If provided, it must be a table holding
the desired options for this key. These options will supersede the instance's
options:

- `ttl`: a number specifying the expiration time period of the cached
  values. The unit is seconds, but accepts fractional number parts, like
  `0.3`. A `ttl` of `0` means the cached values will never expire.
  **Default:** inherited from the instance.
- `neg_ttl`: a number specifying the expiration time period of the cached
  misses (when the L3 callback returns `nil`). The unit is seconds, but
  accepts fractional number parts, like `0.3`. A `neg_ttl` of `0` means the
  cached misses will never expire.
  **Default:** inherited from the instance.
- `resurrect_ttl`: _optional_ number. When specified, `get()` will attempt to
  resurrect stale values when errors are encountered. Errors returned by the L3
  callback (`nil, err`) are considered to be failures to fetch/refresh a value.
  When such return values from the callback are seen by `get()`, and if the
  stale value is still in memory, then `get()` will resurrect the stale value
  for `resurrect_ttl` seconds. The error returned by `get()` will be logged at
  the WARN level, but _not_ returned to the caller. Finally, the `hit_level`
  return value will be `4` to signify that the served item is stale. When
  `resurrect_ttl` is reached, `get()` will once again attempt to run the
  callback. If by then, the callback returns an error again, the value is
  resurrected once again, and so on. If the callback succeeds, the value is
  refreshed and not marked as stale anymore. Due to current limitations within
  the LRU cache module, `hit_level` will be `1` when stale values are promoted
  to the L1 cache and retrieved from there. Lua errors thrown by the
  callback _do not_ trigger a resurrect, and are returned by `get()` as usual
  (`nil, err`). When several workers time out while waiting for the worker
  running the callback (e.g. because the datastore is timing out), then users
  of this option will see a slight difference compared to the traditional
  behavior of `get()`. Instead of returning `nil, err` (indicating a lock
  timeout), `get()` will return the stale value (if available), no error, and
  `hit_level` will be `4`. However, the value will not be resurrected (since
  another worker is still running the callback). The unit for this option is
  seconds, but it accepts fractional number parts, like `0.3`. This option
  **must** be greater than `0`, to prevent stale values from being cached
  indefinitely.
  **Default:** inherited from the instance.
- `shm_set_tries`: the number of tries for the lua_shared_dict `set()`
  operation. When the lua_shared_dict is full, it attempts to free up to 30
  items from its queue. When the value being set is much larger than the freed
  space, this option allows mlcache to retry the operation (and free more slots)
  until the maximum number of tries is reached or enough memory was freed for
  the value to fit.
  **Default:** inherited from the instance.
- `l1_serializer`: _optional_ function. Its signature and accepted values are
  documented under the [get()](#get) method, along with an example. If
  specified, this function will be called each time a value is promoted from the
  L2 cache into the L1 (worker Lua VM). This function can perform arbitrary
  serialization of the cached item to transform it into any Lua object _before_
  storing it into the L1 cache. It can thus avoid your application from
  having to repeat such transformations on every request, such as creating
  tables, cdata objects, loading new Lua code, etc...
  **Default:** inherited from the instance.

The third argument `callback` **must** be a function. Its signature and return
values are documented in the following example:

```lua
-- arg1, arg2, and arg3 are arguments forwarded to the callback from the
-- `get()` variadic arguments, like so:
-- cache:get(key, opts, callback, arg1, arg2, arg3)

local function callback(arg1, arg2, arg3)
    -- I/O lookup logic
    -- ...

    -- value: the value to cache (Lua scalar or table)
    -- err: if not `nil`, will abort get(), which will return `value` and `err`
    -- ttl: ttl for this value - if returned, will override `ttl` or `neg_ttl`
    return value, err, ttl
end
```

This function **can** throw Lua errors as it runs in protected mode. Such
errors thrown from the callback will be returned as strings in the second
return value `err`.

When called, `get()` follows the below logic:

1. query the L1 cache (lua-resty-lrucache instance). This cache lives in the
   Lua VM, and as such, it is the most efficient one to query.
    1. if the L1 cache has the value, return it.
    2. if the L1 cache does not have the value (L1 miss), continue.
2. query the L2 cache (`lua_shared_dict` memory zone). This cache is
   shared by all workers, and is almost as efficient as the L1 cache. It
   however requires serialization of stored Lua tables.
    1. if the L2 cache has the value, return it.
        1. if `l1_serializer` is set, run it, and promote the resulting value
           in the L1 cache.
        2. if not, directly promote the value as-is in the L1 cache.
    2. if the L2 cache does not have the value (L2 miss), continue.
3. create a [lua-resty-lock], and ensures that a single worker will run the
   callback (other workers trying to access the same value will wait).
4. a single worker runs the L3 callback (e.g. performs a database query)
   1. the callback succeeds and returns a value: the value is set in the
      L2 cache, and then in the L1 cache (as-is by default, or as returned by
      `l1_serializer` if specified).
   2. the callback failed and returned `nil, err`:
      a. if `resurrect_ttl` is specified, and if the stale value is still
         available, resurrect it in the L2 cache and promote it to the L1.
      b. otherwise, `get()` returns `nil, err`.
5. other workers that were trying to access the same value but were waiting
   are unlocked and read the value from the L2 cache (they do not run the L3
   callback) and return it.

Example:

```lua
local mlcache = require "mlcache"

local cache, err = mlcache.new("my_cache", "cache_shared_dict", {
    lru_size = 1000,
    ttl      = 3600,
    neg_ttl  = 60
})

local function fetch_user(user_id)
    local user, err = db:query_user(user_id)
    if err then
        -- in this case, get() will return `nil` + `err`
        return nil, err
    end

    return user -- table or nil
end

local user_id = 3

local user, err = cache:get("users:" .. user_id, nil, fetch_user, user_id)
if err then
    ngx.log(ngx.ERR, "could not retrieve user: ", err)
    return
end

-- `user` could be a table, but could also be `nil` (does not exist)
-- regardless, it will be cached and subsequent calls to get() will
-- return the cached value, for up to `ttl` or `neg_ttl`.
if user then
    ngx.say("user exists: ", user.name)
else
    ngx.say("user does not exists")
end
```

This second example is a modification of the above one, in which we apply
some transformation to the retrieved `user` record, and cache it via the
`l1_serializer` callback:

```lua
-- Our l1_serializer, called when a value is promoted from L2 to L1
--
-- Its signature receives a single argument: the item as returned from
-- an L2 hit. Therefore, this argument can never be `nil`. The result will be
-- kept in the L1 cache, but it cannot be `nil`.
--
-- This function can return `nil` and a string describing an error, which
-- will bubble up to the caller of `get()`. It also runs in protected mode
-- and will report any Lua error.
local function load_code(user_row)
    if user_row.custom_code ~= nil then
        local f, err = loadstring(user_row.raw_lua_code)
        if not compiled then
            -- in this case, nothing will be stored in the cache (as if the L3
            -- callback failed)
            return nil, "failed to compile custom code: " .. err
        end

        user_row.f = f
    end

    return user_row
end

local user, err = cache:get("users:" .. user_id,
                            { l1_serializer = load_code },
                            fetch_user, user_id)
if err then
     ngx.log(ngx.ERR, "could not retrieve user: ", err)
     return
end

-- now we can call a function that was already loaded once, upon entering
-- the L1 cache (Lua VM)
user.f()
```

[Back to TOC](#table-of-contents)

peek
----
**syntax:** `ttl, err, value = cache:peek(key)`

Peek into the L2 (`lua_shared_dict`) cache.

The first and only argument `key` is a string, and it is the key to lookup.

This method returns `nil` and a string describing the error upon failure.

Upon success, but if there is no such value for the queried `key`, it returns
`nil` as its first argument, and no error. The same applies to cached misses
looked up with this function.

Upon success, and if there is such a value for the queried `key`, it returns a
number indicating the remaining TTL of the cached value. The third returned
value in that case will be the cached value itself, for convenience.

This method is useful if you want to know whether a value is cached or not. A
value stored in the L2 cache is considered cached, regardless of whether or not
it is also set in the L1 cache of the worker. That is because the L1 cache is
too volatile (as its size unit is a number of slots), and the L2 cache is
still several orders of magnitude faster than the L3 callback anyway.

As its only intent is to take a "peek" into the cache to determine its warmth
for a given value, `peek()` does not count as a query like [get()](#get), and
does not set the value in the L1 cache.

Example:

```lua
local mlcache = require "mlcache"

local cache = mlcache.new("my_cache", "cache_shared_dict")

local ttl, err, value = cache:peek("key")
if err then
    ngx.log(ngx.ERR, "could not peek cache: ", err)
    return
end

ngx.say(ttl)   -- nil because `key` has no value yet
ngx.say(value) -- nil

-- cache the value

cache:get("key", { ttl = 5 }, function() return "some value" end)

-- wait 2 seconds

ngx.sleep(2)

local ttl, err, value = cache:peek("key")
if err then
    ngx.log(ngx.ERR, "could not peek cache: ", err)
    return
end

ngx.say(ttl)   -- 3
ngx.say(value) -- "some value"
```

[Back to TOC](#table-of-contents)

set
---
**syntax:** `ok, err = cache:set(key, opts?, value)`

Unconditionally set a value in the L2 cache and broadcasts an event to other
workers so they can refresh the value from their L1 cache.

The first argument `key` is a string, and is the key under which to store the
value.

The second argument `opts` is optional, and if provided, is identical to the
one of [get()](#get).

The third argument `value` is the value to cache, similar to the return value
of the L3 callback. Just like the callback's return value, it must be a Lua
scalar, a table, or `nil`. If a `l1_serializer` is provided either from the
constructor or in the `opts` argument, it will be called with `value` if
`value` is not `nil`.

On success, the first return value will be `true`.

On failure, this method returns `nil` and a string describing the error.

**Note:** methods such as [set()](#set) and [delete()](#delete) require that
other instances of mlcache (from other workers) refresh the value in their L1
cache. Since OpenResty currently has no built-in mechanism for inter-worker
communication, this module relies on a polling mechanism via a
`lua_shared_dict` memory zone to propagate inter-worker events. If `set()` or
`delete()` are called from a single worker, other workers' mlcache instances
bearing the same `name` must call [update()](#update) before their cache be
requested during the next request, to make sure they evicted their L1 value,
and that the L2 value (which is fresh) will be fetched and returned.

**Note bis:** It is generally considered inefficient to call `set()` on a hot
code path (such as in a request being served by OpenResty). Instead, one should
rely on [get()](#get) and its built-in mutex in the L3 callback. `set()` is
better suited when called occasionally from a single worker, for example upon a
particular event that triggers a cached value to be updated. Once `set()`
updates the L2 cache with the fresh value, other workers will rely on
[update()](#update) to poll the invalidation event. Calling `get()` on those
other workers thus triggers an L1 miss, but the L2 access will hit the fresh
value.

**See:** [update()](#update)

[Back to TOC](#table-of-contents)

delete
------
**syntax:** `ok, err = cache:delete(key)`

Delete a value in the L2 cache and publish an event to other workers so they
can evict the value from their L1 cache.

The first and only argument `key` is the string at which the value is stored.

On success, the first return value will be `true`.

On failure, this method returns `nil` and a string describing the error.

**Note:** methods such as [set()](#set) and [delete()](#delete) require that
other instances of mlcache (from other workers) refresh the value in their L1
cache. Since OpenResty currently has no built-in mechanism for inter-worker
communication, this module relies on a polling mechanism via a
`lua_shared_dict` memory zone to propagate inter-worker events. If `set()` or
`delete()` are called from a single worker, other workers' mlcache instances
bearing the same `name` must call [update()](#update) before their cache be
requested during the next request, to make sure they evicted their L1 value,
and that the L2 value (which is fresh) will be fetched and returned.

**See:** [update()](#update)

[Back to TOC](#table-of-contents)

purge
-----
**syntax:** `ok, err = cache:purge(flush_expired?)`

Purge the content of the cache, in both the L1 and L2 levels. Then publishes
an event to other workers so they can purge their L1 cache as well.

This method recycles the lua-resty-lrucache instance, and calls
[ngx.shared.DICT:flush_all](https://github.com/openresty/lua-nginx-module#ngxshareddictflush_all)
, so it can be rather expensive.

The first and only argument `flush_expired` is optional, but if given `true`,
this method will also call
[ngx.shared.DICT:flush_expired](https://github.com/openresty/lua-nginx-module#ngxshareddictflush_expired)
(with no arguments). This is useful to release memory claimed by the L2 (shm)
cache if needed.

On success, the first return value will be `true`.

On failure, this method returns `nil` and a string describing the error.

**Note:** methods such as [set()](#set) and [delete()](#delete) require that
other instances of mlcache (from other workers) refresh the value in their L1
cache. Since OpenResty currently has no built-in mechanism for inter-worker
communication, this module relies on a polling mechanism via a
`lua_shared_dict` memory zone to propagate inter-worker events. If `set()` or
`delete()` are called from a single worker, other workers' mlcache instances
bearing the same `name` must call [update()](#update) before their cache be
requested during the next request, to make sure they evicted their L1 value,
and that the L2 value (which is fresh) will be fetched and returned.

**See:** [update()](#update)

[Back to TOC](#table-of-contents)

update
------
**syntax:** `ok, err = cache:update()`

Poll and execute pending cache invalidation events published by other workers.

Methods such as [set()](#set) and [delete()](#delete) require that other
instances of mlcache (from other workers) refresh the value in their L1 cache.
Since OpenResty currently has no built-in mechanism for inter-worker
communication, this module bundles an "off-the-shelf" IPC library to propagate
inter-worker events. If the bundled IPC library is used, the `lua_shared_dict`
specified in the `ipc_shm` option **must not** be used by other actors than
mlcache itself.

This method allows a worker to update its L1 cache (by purging values
considered stale due to an other worker calling `set()` or `delete()`) before
processing a request.

A typical design pattern is to call `update()` **only once** before each
request processing. This allows your hot code paths to perform a single shm
access in the best case scenario: no invalidation events were received, all
`get()` calls will hit in the L1 cache. Only on a worst case scenario (`n`
values were evicted by another worker) will `get()` access the L2 or L3 cache
`n` times. Subsequent requests will then hit the best case scenario again,
because `get()` populated the L1 cache.

For example, if your workers make use of [set()](#set) or [delete()](#delete)
anywhere in your application, call `update()` at the entrance of your hot code
path, before using `get()`:

```
http {
    listen 9000;

    location / {
        content_by_lua_block {
            local cache = ... -- retrieve mlcache instance

            -- make sure L1 cache is evicted of stale values
            -- before calling get()
            local ok, err = cache:update()
            if not ok then
                ngx.log(ngx.ERR, "failed to poll eviction events: ", err)
                -- /!\ we might get stale data from get()
            end

            -- L1/L2/L3 lookup (best case: L1)
            local value, err = cache:get("key_1", nil, cb1)

            -- L1/L2/L3 lookup (best case: L1)
            local other_value, err = cache:get(key_2", nil, cb2)

            -- value and other_value are up-to-date because:
            -- either they were not stale and directly came from L1 (best case scenario)
            -- either they were stale and evicted from L1, and came from L2
            -- either they were not in L1 nor L2, and came from L3 (worst case scenario)
        }
    }

    location /delete {
        content_by_lua_block {
            local cache = ... -- retrieve mlcache instance

            -- delete some value
            local ok, err = cache:delete("key_1")
            if not ok then
                ngx.log(ngx.ERR, "failed to delete value from cache: ", err)
                return ngx.exit(500)
            end

            ngx.exit(204)
        }
    }

    location /set {
        content_by_lua_block {
            local cache = ... -- retrieve mlcache instance

            -- update some value
            local ok, err = cache:set("key_1", nil, 123)
            if not ok then
                ngx.log(ngx.ERR, "failed to set value in cache: ", err)
                return ngx.exit(500)
            end

            ngx.exit(200)
        }
    }
}
```

**Note:** you **do not** need to call `update()` to refresh your workers if
they never call `set()`or `delete()`. When workers only rely on `get()`, values
expire naturally from the L1/L2 caches according to their TTL.

**Note bis:** this library was built with the intent to use a better solution
for inter-worker communication as soon as one emerges. In future versions of
this library, if an IPC library can avoid the polling approach, so will this
library. `update()` is only a necessary evil due to today's Nginx/OpenResty
"limitations". You can however use your own IPC library by use of the
`opts.ipc` option when creating your mlcache instance.

[Back to TOC](#table-of-contents)

# Changelog

See [CHANGELOG.md](CHANGELOG.md).

[Back to TOC](#table-of-contents)

# License

Work licensed under the MIT License.

[Back to TOC](#table-of-contents)

[lua-resty-lock]: https://github.com/openresty/lua-resty-lock
[lua-resty-lrucache]: https://github.com/openresty/lua-resty-lrucache
[lua_shared_dict]: https://github.com/openresty/lua-nginx-module#lua_shared_dict

[badge-travis-url]: https://travis-ci.org/thibaultcha/lua-resty-mlcache
[badge-travis-image]: https://travis-ci.org/thibaultcha/lua-resty-mlcache.svg?branch=master
