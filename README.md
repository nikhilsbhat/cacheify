# Cacheify

Simple cache plugin middleware caches responses on disk.

Based on the original plugin-simplecache, but with some significant performance improvements
- Cache hits are now up to 13× faster and use 45% less memory, with large payloads seeing over 90% latency reduction.
- Streamlined miss handling and buffer reuse (-45–60% memory use on large bodies)
- Reduced allocations per operation by ~20% overall
- Improved concurrent hit performance (no longer contended)
- Simplified and accelerated cache key generation (-75–88% time)
- Achieved ~36% faster benchmarks overall and ~60% higher throughput

## Platform Support

**Unix/Linux/macOS only** - This plugin is not compatible with Windows due to Traefik's Yaegi interpreter security restrictions on `unsafe` and `syscall` packages required for atomic file operations on Windows.

## Configuration

To configure this plugin you should add its configuration to the Traefik dynamic configuration as explained [here](https://docs.traefik.io/getting-started/configuration-overview/#the-dynamic-configuration).
The following snippet shows how to configure this plugin with the File provider in TOML and YAML: 

Static:

```toml
[experimental.plugins.cache]
  modulename = "github.com/ciaranj/cacheify"
  version = "v0.0.1"
```

Dynamic:

```toml
[http.middlewares]
  [http.middlewares.my-cache.plugin.cache]
    path = "/some/path/to/cache/dir"
    maxSize = "10g"
```

```yaml
http:
  middlewares:
   my-cache:
      plugin:
        cache:
          path: /some/path/to/cache/dir
          maxSize: 10g
```

### Options

#### Path (`path`)

The base path that files will be created under. This must be a valid existing
filesystem path.

#### Max Expiry (`maxExpiry`)

*Default: 300*

The maximum number of seconds a response can be cached for. The 
actual cache time will always be lower or equal to this.

#### Cleanup (`cleanup`)

*Default: 600*

The number of seconds to wait between cache cleanup runs.

#### Max Size (`maxSize`)

*Default: unset*

An optional disk usage limit for the cache, using nginx-style size suffixes like
`10g`, `500m`, `128k`, or raw bytes such as `1048576`. When set, cacheify will
evict the oldest cache files during its cleanup pass until the total cache size
falls back under the limit. If omitted or set to `0`, size-based eviction is
disabled.
	
#### Add Status Header (`addStatusHeader`)

*Default: true*

This determines if the cache status header `Cache-Status` will be added to the
response headers. This header can have the value:

* `hit`: response served directly from cache
* `miss`: no cache entry was available, so the response came from upstream
* `bypass`: request/response was intentionally not cached
* `expired`: an expired cache entry existed and the response was refetched
* `updating`: another request populated the cache and this request reused that result
* `error`: cache lookup/write failed and the response fell back to upstream

#### Include Query Parameters in Cache Key (`queryInKey`)
*Default: true*

This determines whether the query parameters on the url form part of the key used for storing cacheable requests.

#### Max Header Pairs (`maxHeaderPairs`)
*Default: 255*

The maximum number of header key-value pairs allowed in cached responses. This prevents disk bloat attacks from responses with excessive headers. Multi-value headers (e.g., multiple `Set-Cookie` headers) count as separate pairs.

#### Max Header Key Length (`maxHeaderKeyLen`)
*Default: 100*

The maximum length in bytes for header keys (names). This prevents disk bloat from maliciously long header names. Standard HTTP header names are typically 10-30 bytes.

#### Max Header Value Length (`maxHeaderValueLen`)
*Default: 8192*

The maximum length in bytes for header values. This prevents disk bloat from oversized cookies, tokens, or other header values. The default allows for large JWTs and session cookies while preventing abuse.

#### Strip Response Cookies (`stripResponseCookies`)
*Default: true*

If true (the default) cacheify will remove any 'Set-Cookie' headers from any cacheable responses (including the original request.)

#### Update Timeout (`updateTimeout`)
*Default: 30*

The number of seconds to wait for another request to complete a cache update before timing out and fetching from upstream independently. This prevents requests from waiting indefinitely if an upstream server hangs during a cache miss. When multiple requests arrive for the same uncached resource, the first request fetches from upstream while subsequent requests wait for completion. If the timeout is exceeded, waiting requests will proceed to fetch from upstream themselves rather than block indefinitely.

## Release History
### v1.0.0
#### Bugfixes
* Downstream (traefik) disconnects & timeouts could lead to 0 byte bodies being cached
* There was a memory leak identified in the lock management, the fix has incurred a minor performance penalty, but we are still largely IO bound.
#### Breaking change notice
This release introduces a potentially breaking change in behaviour. Prior to 1.0.0 there was a bug that meant responses that should be 'heuristically' cached, that is responses that don't explicitly decline caching behaviours but that should by default (according to the specification) be cached, were not being cached.

This release changes the behaviour so that these default 'heuristically' cacheable responses will now be cached, the heuristic used is currently whatever you have set in your maxExpiry setting.

This will likely result in a significantly higher number of responses being cached in v1.0.0 than previously and is not currently a configurable behaviour. If there is demand for that we can introduce it.
