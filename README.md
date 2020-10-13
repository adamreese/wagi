# WAGI: Web Assembly Gateway Interface

_WAGI is the easiest way to get started doing cloud-side WASM web apps._

**WARNING:** This is experimental code put together on a whim.
It is not considered production-grade by its developers, neither is it "supported" software.
This is a project we wrote to demonstrate another way to use WASI.

> DeisLabs is experimenting with many WASM technologies right now.
> This is one of a multitude of projects (including [Krustlet](https://github.com/deislabs/krustlet))
> designed to test the limits of WebAssembly as a cloud-based runtime.

## tl;dr

WAGI allows you to run WebAssembly WASI binaries as HTTP handlers.
Write a "command line" application that prints a few headers, and compile it to WASM32-WASI.
Add an entry to the `modules.toml` matching URL to WASM module.
That's it.

You can use any programming language that can compile to WASM32-WASI.

## What Is WebAssembly Gateway Interface (WAGI)

Like fashion, all technologies eventually make a comeback.
WAGI (pronounced "waggy") is an implementation of CGI for WebAssembly and
WebAssembly System Interface (WASI).

In the current specification, WASI does not have a networking layer.
It has environment variables and file handles, but no way of creating a server.
Further, current WASM implementations are single-threaded and have no support for concurrency.
So it would be difficult to write a decent web server implementation anyway.

But these problems are nothing new.
In the 1990s, as the web was being created, a standard arose to make it easy to attach scripts to a web server, thus providing dynamic server-side logic.
This standard was the [Common Gateway Interface](https://tools.ietf.org/html/rfc3875), or CGI.
CGI made it dead simple to write stand-alone programs in any language, and have them answer Web requests.
WASM + WASI is suited to implementing the same features.

WAGI provides an HTTP server implementation that can dynamically load and execute WASM modules using the same techniques as CGI scripts.
Headers are placed in environment variables.
Query parameters, when present, are sent in as command line options.
Incoming HTTP payloads are sent in via STDIN.
And the response is simply written to STDOUT.

Because the system is so simple, writing and testing WAGI is simple, regardless of the language you choose to write in.
You can even execute WAGIs without a server runtime.

WAGI is designed to be higher in security than CGI.
Thus WAGIs have more security restrictions.

- They cannot access many things (including the filesystem) without explicit access grants.
- They cannot make outbound network connections.
- They cannot execute other executables on the system.
- They cannot access arbitrary environment variables--only ones that are explicitly passed in.

In the future, as WASI matures, we will relax the restrictions on outbound networking.

## Getting Started

To run the WAGI server, use `cargo run`.
You can also `cargo build` WAGI and run it as a static binary.

If you want to understand the details, read the [Common Gateway Interface 1.1](https://tools.ietf.org/html/rfc3875) specification.
While this is not an exact implementation, it is very close.
See the "Differences" section below for the differences.

### Examples and Demos

- [env_wagi](https://github.com/deislabs/env_wagi): Dump the environment that WAGI sets up, including env vars and args.

## Configuring the WAGI Server

The WAGI server uses a `modules.toml` file to point to the WAGI modules that can be executed.
(A WAGI module is just a WASM+WASI module that prints its output correctly.)

Here is an example `modules.toml`:

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
```
### TOML fields

- Top-level fields
  - Currently none
- The `[[module]]` list: Each module starts with a `[[module]]` header. Inside of a module, the following fields are available:
  - `route`: The path that is appended to the server URL to create a full URL (e.g. `/foo` becomes `https://example.com/foo`)
  - `module`: The absolute path to the module on the file system
  - `environment`: A list of string/string environment variable pairs.
  - `repository`: RESERVED

## Writing WAGI Modules

A WAGI module is a WASM binary compiled with WASI support.

A module must have a `_start` method. Most of the time, that is generated by the compiler.
More often, an implementation of a WAGI module looks like this piece of pseudo-code:

```javascript
function main() {
    println("Content-Type: text/plain") // Required header
    println("")                         // Empty line is also require
    println("hello world")              // The body
}
```

Here is the above written in Rust:

```rust
fn main() {
    println!("Content-Type: text/plain\n");
    println!("hello world");
}
```

In Rust, you can compile the above with `cargo build --target wasm32-wasi --release` and have a WAGI module ready to use!

And here is an [AssemblyScript](https://www.assemblyscript.org) version:

```typescript
import "wasi";
import { Console } from "as-wasi";

Console.log("content-type: text-plain");
Console.log(""); // blank line separates headers from body.
Console.log("hello world");
```

Note that the AssemblyScript compiler generates the function body wrapper for you.
For more, check out the AssemblyScript WASI [docs](https://wasmbyexample.dev/examples/wasi-hello-world/wasi-hello-world.assemblyscript.en-us.html).

## Differences from CGI 1.1

- We use UTF-8 instead of ASCII.
- WAGIs are not handled as "processes", they are executed internally with multi-threading.
- WAGIs do _not_ have unrestricted access to the underlying OS or filesystem.
    * If you want to give a WAGI access to a portion of the filesystem, you must configure the WAGI's `wagi.toml` file
    * WAGIs cannot make outbound network connections
    * Some CGI env vars are rewritten to remove local FS information
- WAGIs have a few extra CGI environment variables, prefixed with `X_`.
- A `location` header from a WAGI must return a full URL, not a path. (CGI supports both)
    * This will set the status code to `302 Found` (per 6.2.4 of the CGI specification)
    * If `status` is returned AFTER `location`, it will override the status code
- WAGI does NOT support NPH (Non-Parsed Header) mode
- The value of `args` is NOT escaped for borne-style shells (See section 7.2 of CGI spec)

It should be noted that while the daemon (the WAGI server) runs constantly, both the `modules.toml` and the `.wasm` file are loaded for each request, much as they were for CGI.
In the future, the WAGI server may cache the WASM modules to speed loading.
But in the near term, we are less concerned with performance and more concerned with debugging.