---
title: Caching
description: Cache data in Redis with frappe.cache, read documents with get_cached_doc and get_cached_value, and use the request_cache, site_cache and redis_cache decorators.
version: develop
sources:
  - frappe/utils/redis_wrapper.py
  - frappe/utils/caching.py
  - frappe/cache_manager.py
  - frappe/model/document.py
  - frappe/__init__.py
  - frappe/commands/utils.py
  - frappe/hooks.py
---

# Caching

Use a cache when your code reads the same data many times and the data changes rarely: settings, master records, computed lookups. Frappe has a Redis cache shared by all processes of the bench (`frappe.cache`), helpers that cache whole documents in it, and decorators that cache function results per request, per process or in Redis. Pick the smallest scope that works, and always plan how the cache gets cleared.

## Scopes

| Tool | Stored in | Lives | Shared across processes |
|---|---|---|---|
| `@request_cache` | `frappe.local.request_cache` | One request or job | No |
| `@site_cache` | Process memory | Until `ttl` or `clear_cache()` | No |
| `frappe.cache` / `@redis_cache` | Redis (`redis_cache` URL) | Until expiry or delete | Yes |
| `frappe.get_cached_doc` / `get_cached_value` | Redis | 1 hour, or until the document is saved | Yes |
| `frappe.client_cache` | Process memory + Redis | 10 minutes local, invalidated by Redis | Yes (internal use) |

## frappe.cache

`frappe.cache` is a `RedisWrapper` (a subclass of `redis.Redis`). Values are pickled, so you can store dicts, lists and documents.

```python
frappe.cache.set_value("sample_app:rates", {"Hardware": 10}, expires_in_sec=3600)
rates = frappe.cache.get_value("sample_app:rates")

# compute and store the value when it is missing
rates = frappe.cache.get_value("sample_app:rates", generator=load_rates)

frappe.cache.delete_value("sample_app:rates")
frappe.cache.delete_keys("sample_app:")  # every key that starts with the prefix
```

Keys are namespaced by site. `frappe.cache.make_key("k")` returns `<db_name>|k`. Options on most methods:

- `user=True` adds `user:<current user>:` to the key. Pass a user ID to use another user.
- `shared=True` uses the key as given, with no site prefix. All sites on the bench see it.

| Method | Description |
|---|---|
| `set_value(key, val, user=None, expires_in_sec=None, shared=False)` | Store a value. Returns the size in bytes. |
| `get_value(key, generator=None, user=None, expires=False, shared=False)` | Read a value. Calls `generator` on a miss and stores the result. |
| `delete_value(keys, user=None, make_keys=True, shared=False)` | Delete one key or a list of keys. |
| `delete_keys(key)` | Delete all keys that start with `key`. |
| `get_keys(key)` | List keys that start with `key`. |
| `exists(*names, user=None, shared=None)` | Count of keys that exist. |
| `expire_key(key, time)` | Set a TTL in seconds. |
| `hset(name, key, value)`, `hget(name, key, generator=None)`, `hdel(name, keys)`, `hgetall(name)` | Hash operations, with the site prefix. |
| `lpush`, `rpush`, `lpop`, `rpop`, `llen`, `lrange`, `ltrim`, `sadd`, `srem`, `smembers` | List and set operations. |

`get_value` also keeps the value in `frappe.local.cache` for the rest of the request. A second read in the same request does not go to Redis. Pass `expires=True` for keys with a TTL so the local copy is not kept.

If Redis is not reachable, `set_value` and `get_value` do not raise. `get_value` returns `None`, or the generator result.

!!! note "Deprecated"
    `frappe.cache()` (called as a function) is deprecated. Use `frappe.cache`.

## Cached documents

```python
settings = frappe.get_cached_doc("System Settings")
item = frappe.get_cached_doc("Sample Item", "Laptop")
rate = frappe.get_cached_value("Sample Item", "Laptop", "rate")
values = frappe.get_cached_value("User", "user1@example.com", ["first_name", "user_type"], as_dict=True)
time_zone = frappe.get_single_value("System Settings", "time_zone")
```

- `get_cached_doc` takes the same arguments as `frappe.get_doc`. It stores the document in Redis for 1 hour under `document_cache::<doctype>::<name>`.
- Saving or deleting the document clears its cache entry (`Document.clear_cache`). The entry is cleared again after commit and after rollback.
- Clear it yourself with `frappe.clear_document_cache(doctype, name)`. Without `name`, it clears all cached documents of the DocType.
- `get_cached_value` returns `None` if the document does not exist.
- `frappe.get_single_value(doctype, fieldname)` reads a field of a Single DocType through `get_cached_value`.
- These functions do not check permissions.

!!! warning
    `frappe.db.set_value` and `frappe.db.set_single_value` clear the document cache. Raw SQL (`frappe.db.sql`) and `frappe.qb` updates do not. After such writes, call `frappe.clear_document_cache`.

## Decorators

All three are in `frappe.utils.caching`. `frappe.request_cache` and `frappe.site_cache` are aliases.

### request_cache

```python
from frappe.utils.caching import request_cache


@request_cache
def get_item_rate(item: str) -> float:
	return frappe.db.get_value("Sample Item", item, "rate")
```

Calls with the same arguments in one request or job run once. Arguments must be hashable. Unhashable arguments skip the cache.

### site_cache

```python
from frappe.utils.caching import site_cache


@site_cache(ttl=300, maxsize=128)
def get_category_map() -> dict:
	return dict(frappe.get_all("Sample Item", fields=["name", "category"], as_list=True))


get_category_map.clear_cache()  # clear for all sites in this process
```

The key includes the site, so sites do not share results. The cache is per process: each web worker and background worker has its own copy, and `clear_cache()` only clears the current process. `frappe.clear_cache()` clears all `site_cache` entries in the current process. When `maxsize` is reached, the oldest entry is removed.

### redis_cache

```python
from frappe.utils.caching import redis_cache


@redis_cache(ttl=600)
def get_open_order_count(customer: str) -> int:
	return frappe.db.count("Sample Order", {"customer_name": customer, "status": "Open"})


get_open_order_count.clear_cache()
```

| Argument | Default | Description |
|---|---|---|
| `ttl` | 3600 | Seconds to keep each result. |
| `user` | `None` | `True` caches one result for each session user. |
| `shared` | `False` | `True` shares the results across sites. |

The cache key is `<module>.<function>::<hash of arguments>`. `clear_cache()` deletes every result of the function. A `None` result is cached too.

### http_cache

`@http_cache(public=False, max_age=None, stale_while_revalidate=None)` sets a `Cache-Control` header on GET responses of a whitelisted method. It does not cache on the server.

## client_cache

`frappe.client_cache` keeps hot keys in process memory and uses Redis client-side caching to invalidate them. Frappe uses it for hooks, schema and settings. The docstring says it is internal and can change without notice. Use `frappe.cache` in apps. It has `get_value`, `set_value`, `delete_value`, `delete_keys` and `get_doc`. The local TTL is 10 minutes and the default size is 1024 keys. `client_cache_max_bytes` in site config limits its memory.

## Clear the cache

```bash
bench --site frappe.localhost clear-cache
bench --site frappe.localhost clear-website-cache
```

```python
frappe.clear_cache()                       # all site keys, site_cache, client_cache
frappe.clear_cache(doctype="Sample Item")  # DocType meta and related caches
frappe.clear_cache(user="user1@example.com")  # user caches (roles, permissions, defaults)
```

A full `frappe.clear_cache()` deletes all Redis keys of the site except keys that match the `persistent_cache_keys` hook. Then it runs every `clear_cache` hook.

```python
# sample_app/hooks.py
persistent_cache_keys = ["sample_app:rates*"]  # kept by frappe.clear_cache()
clear_cache = "sample_app.cache.clear_sample_cache"  # called by frappe.clear_cache()
```

## Pitfalls

- Never cache data that depends on permissions in a key that other users can read. Use `user=True`.
- Changes from `site_cache` do not reach other processes. Do not use it for data that users edit.
- Cached documents are copies. Changing a cached document does not change the database. Use `frappe.get_doc` when you will save.
- Do not cache large values in `client_cache`. It is sized for values of about 4 KB.

## Related

- [Performance](../testing-and-performance/performance.md)
- [Database](../database/database.md)
- [Configuration](../operations/configuration.md)
- [Hooks reference](../../reference/configuration/hooks.md)
