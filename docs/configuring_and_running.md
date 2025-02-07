# Getting Started with WAGI

This guide covers configuring and running the WAGI server, as well as loading a WebAssembly module.
It assumes you have already [installed](installation.md) WAGI.

This guide begins with starting the WAGI server, then covers the `modules.toml` and `cache.toml` configuration files.

## Running WAGI

The `wagi` server is run from the command line. It has a few flags:

- `-c`|`--config` (REQUIRED): The path to a `modules.toml` configuration
- `--cache`: The path to an optional `cache.toml` configuration file (see the caching section below)
- `-l`|`--listen`: The IP address and port to listen on. Default is 127.0.0.1:3000

At minimum, to start WAGI, run a command that looks like this:

```console
$ wagi -c examples/modules.toml
=> Starting server
(load_routes) instantiation time for module examples/hello.wat: 101.840297ms
(load_routes) instantiation time for module examples/hello.wasm: 680.518671ms
```

To start from source, use `cargo run -- -c examples/modules.toml`.

## The `modules.toml` Configuration File

WAGI requires a TOML-formatted configuration file that details which modules should be loaded.
By convention, this file is called `modules.toml`.

In a nutshell, these are the fields that `modules.toml` supports.

- Top-level fields
  - `default_host`: By default, WAGI answers only to the hostname `localhost`. This allows you to specify a different default domain.
- The `[[module]]` list: Each module starts with a `[[module]]` header. Inside of a module, the following fields are available:
  - `route` (REQUIRED): The path that is appended to the server URL to create a full URL (e.g. `/foo` becomes `https://example.com/foo`)
  - `module` (REQUIRED): A module reference. See Module References below.
  - `environment`: A list of string/string environment variable pairs.
  - `repository`: RESERVED for future use
  - `entrypoint` (default: `_start`): The name of the function within the module. This will directly execute that function. Most WASM/WASI implementations create a `_start` function by default. An example of a module that declares 3 entrypoints can be found [here](https://github.com/technosophos/hello-wagi).
  - `host`: The hostname for this app. If set, the only clients that send this name in the `HOST` http header will be directed to this handler.

Here is a brief example of a `modules.toml` file that declares two routes:

```toml
[[module]]
# Example executing a Web Assembly Text file
route = "/"
module = "examples/hello.wat"

[[module]]
# Example running a WASM file.
route = "/hello/..."
module = "examples/hello.wasm"
```

Each `[[module]]` section in the `modules.toml` file is responsible for mapping a route (the path part of a URL) to an executable piece of code.

The two required directives for a module section are:

- `route`: The path-portion of a URL
- `module`: A reference to the WebAssembly module to execute

Routes are paths relative to the WAGI HTTP root. Assuming the routes above are running on a server whose domain is `example.com`:

- The `/` route handles traffic to `http://example.com/` (or `https://example.com/`)
- A route like `/hello` would handle traffic to `http://example.com/hello`
- The route `/hello/...` is a special wildcard route that handles any traffic to `http://example.com/hello` or a subpath (like `http://example.com/hello/today/is/a/goo/day`)

### Module References

A module reference is a URL. There are three supported module reference schemes:

- `file://`: A path to a `.wasm` or `.wat` file on the filesystem. We recommend using absolute paths beginning with `file://`. Right now, there is legacy support for absolute and relative paths without the `file://` prefix, but we discourge using that. Relative paths will be resolved from the current working directory in which `wagi` was started.
- `bindle:`: A reference to a Bindle. This will be looked up in the configured Bindle server. Example: `bindle:example.com/foo/bar/1.2.3`. Bindle URLs do not ever have a `//` after `bindle:`.
- `oci`: A reference to an OCI image in an OCI registry. Example: `oci:foo/bar:1.2.3` (equivalent to the Docker image `foo/bar:1.2.3`). OCI URLs should not need `//` after `oci://`.

#### Volume Mounting

In addition to the required directives, the `[[module]]` sections support several other directives.
One of these is the `volume` directive, which mounts a local directory into the module.

By default, Wasm modules in WAGI have no ability to access the host filesystem.
That is, a Wasm module cannot open `/etc/` and read the files there, even if the `wagi` server has access to `/etc/`.
In WAGI, modules are considered untrusted when it comes to accessing resources on the host.
But it is definitely the case that code sometimes needs access to files.
For that reason, WAGI provides the `volumes` directive.

Here is an example of providing a volume:

```toml
[[module]]
route = "/bar"
module = "/path/to/bar.wasm"
# You can give WAGI access to particular directories on the filesystem.
volumes = {"/path/inside/wasm": "/path/on/host"}
```

In this case, the `volumes` directive tells WAGI to expose the contents of `/path/on/host` to the `bar.wasm` module.
But `bar.wasm` will see that directory as `/path/inside/wasm`. Importantly, it will not be able to access any other parts of the filesystem. Fo example, it will not see anything on the path `/path/inside`. It _only_ has access to the paths specified
in the `volumes` directive.

#### Environment Variables

Similarly to volumes, by default a WebAssembly module cannot access the host's environment variables.
However, WAGI provides a way for you to pass in environment variables:

```toml
[[module]]
route = "/hello"
module = "/path/to/hello.wasm"
# You can put static environment variables in the TOML file
environment.TEST_NAME = "test value"
```

In this case, the environment variable `TEST_NAME` will be set to `test value` for the `hello.wasm` module.
When the module starts up, it will be able to access the `TEST_NAME` variable.

Note that while the module will not be able to access the host environment variables, WAGI does provide a wealth of other environment variables. See [Environment Variables](environment_variables.toml) for details.

#### Entrypoint

By default, a WASM WASI module has a function called `_start()`.
Usually, this function is created at compilation time, and typically it just calls the `main()`
function (this is a detail specific to the language in which the code was written).

Sometimes, though, you may want to have WAGI invoke another function.
This is what the `entrypoint` directive is for.

The following example shows loading the same module at three different paths, each time
invoking a different function:

```toml
# With no `entrypoint`, this will invoke `_start()`
[[module]]
route = "/hello"
module = "/path/to/bar.wasm"

[[module]]
route = "/entrypoint/hello"
module = "/path/to/bar.wasm"
entrypoint = "hello"  # Executes the `hello()` function in the module (instead of `_start`)

[[module]]
route = "/entrypoint/goodbye"
module = "/path/to/bar.wasm"
entrypoint = "goodbye  # Executes the `goodbye()` function in the module (instead of `_start`)
```

### Host Mapping

HTTP applications often support "virtual hosting" in which one HTTP server can listen for multiple hostnames, and then use the hostname to direct traffic to the correct handler.

WAGI supports this with the `host` directive:

```toml
# With no `entrypoint`, this will invoke `_start()`
[[module]]
route = "/"
module = "/path/to/hello.wasm"
host = "example.com"

[[module]]
route = "/"
module = "/path/to/goodbye.wasm"
host = "another.example.com"
```

The above shows two modules that each declare that they listen for the route `/`.
However, they declare different `host` directives.
WAGI will differentiate between these two requests based on the `Host` header in the incoming HTTP requests.

If a client requests `https://example.com/`, WAGI will execute the `hello.wasm` module.
If the client requests `https://another.example.com`, WAGI will execute the `goodbye.wasm` module.
If no matching host is found, WAGI will serve an error.

To provide a default host, set the `default_host` field at the top of your `modules.toml`.
Any module that does not declare a `host` will automatically listen (only) on the
`default_host`. If no `default_host` is specified and a module does not provide a `host`, then
WAGI will use `localhost:3000` as the default host.

```toml
default_host = "my.example.com"

# With no `entrypoint`, this will invoke `_start()`
[[module]]
route = "/"
module = "/path/to/hello.wasm"
host = "example.com"

// This is the default host:
[[module]]
route = "/"
module = "/path/to/goodbye.wasm"
```

Currently, there is no support for "wildcard" domain names. That is, you cannot specify `*.example.com` to match any domain that ends with `.example.com`.

### A Large Example

Here is an example `modules.toml` that exercises the features discussed above:

```toml
[[module]]
route = "/"
module = "/absolute/path/to/root.wasm"

[[module]]
route = "/foo"
module = "/path/to/foo.wasm"

[[module]]
# The "/..." suffix means this will match /bar and its subpaths, like /bar/a/b/c
route = "/bar/..."
module = "/path/to/bar.wasm"
# You can give WAGI access to particular directories on the filesystem.
volumes = {"/path/inside/wasm": "/path/on/host"}
# You can also put static environment variables in the TOML file
environment.TEST_NAME = "test value" 

[[module]]
# You can also execute a WAT file directly
route = "/hello"
module = "/path/to/hello.wat"


# You can declare custom handler methods as 'entrypoints' to the module.
# Here we have two module entries that use the same module, but call into
# different entrypoints.
[[module]]
route = "/entrypoint/hello"
module = "/path/to/bar.wasm"
entrypoint = "hello"  # Executes the `hello()` function in the module (instead of `_start`)

[[module]]
route = "/entrypoint/goodbye"
module = "/path/to/bar.wasm"
entrypoint = "goodbye  # Executes the `goodbye()` function in the module (instead of `_start`)
```

## Enabling Caching

To enable the [Wasmtime cache](https://docs.wasmtime.dev/cli-cache.html), which caches the result of the compilation
of a WebAssembly module, resulting in improved instantiation times for modules, you must create a `cache.toml` file
with the following structure, and point the WAGI binary to it:

```toml
[cache]
enabled = true
directory = "<absolute-path-to-a-cache-directory>"
# optional
# see more details at https://docs.wasmtime.dev/cli-cache.html
cleanup-interval = "1d"
files-total-size-soft-limit = "10Gi"
```

To start WAGI with caching enabled, use the `--cache` flag.
For example, `cargo run -- --config path/to/modules.toml --cache path/to/cache.toml`.

The WAGI server now prints the module instantiation time, so you can choose whether caching helps for your modules.

## What's Next?

Next, read about [Writing Modules](writing_modules.md) for WAGI.