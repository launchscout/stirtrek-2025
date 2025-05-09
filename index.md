---
marp: true
style: |

  section h1 {
    color: #6042BC;
  }

  section code {
    background-color: #e0e0ff;
  }

  footer {
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: 100px;
  }

  footer img {
    position: absolute;
    width: 120px;
    right: 20px;
    top: 0;

  }
  section #title-slide-logo {
    margin-left: -60px;
  }
---

# WebAssembly beyond the browser: a better way to build extensible software
## Chris Nelson
@superchris.launchscout.com (BlueSky)
github.com/superchris
![h:200](/images/full-color.png#title-slide-logo)

---

# Agenda
## WebAssembly Components
- What and Why
- Building and running
- Use cases
- Future

---

# What is WebAssembly?
- VM instruction format
- Near-native execution speed
- Memory-safe, sandboxed execution environment
- Designed as a compilation target for languages like C, C++, Rust
- Started in the browser, but growing usage on the server
- Platform and language independent

---

# WebAssembly Core (1.0)
- Supports only numeric types
- Linear memory
  - Your job to manage
  - Your job to figure out what to put there

---

# Saying hello from WebAssembly
- Let's allocate some shared memory
- passing pointers and offsets and lengths, oh my!
- oh yeah, we should figure out what encoding to use
- Are you sad yet?

---

# WebAssembly Components
- Higher-level abstraction on top of core WebAssembly
- Enables language-agnostic type system
- Establishes canonical ABI
  - how to map high level types to memory
- Enables language-agnostic communication

---

# WIT (WebAssembly Interface Types)
- Interface Definition Language
- Describes components' interfaces
- Defines data types and functions

---

# WIT structure
- package - top level container of the form `namespace:name@version` 
- worlds - specifies the "contract" for a component and contains
  - exports - functions (or interfaces) provided by a component
  - imports - functions (or interfaces) provided by a component
- interfaces - named group of types and functions
- types

---

# WIT types
- Primitive types
  - u8, u16, u32, u64, s8, s16, s32, s64, f32, f64
  - bool, char, string
- Compound types
  - lists, options, results, tuples
- User Defined Types
  - records, variants, enums, flags, 
- Resources
  - references to things that live outside the component

---

# Language support
- Rust
- Javascript
- C/C++
- C# (.NET)
- Go
- Python
- Moonbit
- Elixir (host only)
- Ruby (host only)

---

# Definitions
- **bindings** - language specific stubs generated from WIT
- **exports** - functions that a component provides
- **imports** - functions that a component can call
- **Hosts** - Code which calls into webassembly components
- **Guests** - Code which produces webassembly components to be called from Hosts
- **Runtimes** - Provides all required services to access the outside world
  - browser (jco)
  - server (wasmtime)

---

# Hello World WIT
```wit
package component:hello-world;

world hello-world {
    export greet: func(greetee: string) -> string;
}
```

---

# Implementing Hello world in Rust
## Step 1: generate bindings
```
cargo component bindings
```

---
# Implement the component
```rust
#[allow(warnings)]
mod bindings;

use bindings::Guest;

struct Component;

impl Guest for Component {
    /// Say hello!
    fn greet(greetee: String) -> String {
        format!("Hello from Rust, {}!", greetee)
    }
}

bindings::export!(Component with_types_in bindings);
```

---

# Build
```
cargo component build
```

---

# Now we need a runtime...
## Guess what: we already have one!

---

# Webcomponents in the browser with jco
- javascript toolchain for WebAssembly Component runtime
- handles both hosting and guesting
- `jco componentize` creates guest components from javascript
- `jco transpile` creates an ES module wrapper to host a component

---

# [Demo time!](./demo1.html)
- change some code
- run our build
- show the html
- see it in the browser

---

# Let's talk imports
## Where do they come from?
- Provided by host
- Provided by another component
- **Component composition**

---

# Hello, with imports
```wit
package component:composed-hello-world;

world composed-hello-world {
  import additional-greeting: func() -> string;
  export greet: func(greetee: string) -> string;
}
```

---

## A component which exports `additional-greeting`
```wit
package local:additional-greeting;

world additional-greeting {
  export additional-greeting: func() -> string;
}
```
---

# Implementing `additional-greeting` in Go
### Generating bindings
```sh
wkg wit fetch
wkg wit build
go tool wit-bindgen-go generate --world additional-greeting --out internal ./local:additional-greeting@0.0.1.wasm
```

---
```golang
package main

import (
	"additionalgreeting/internal/local/additional-greeting/additional-greeting"
)

func init() {
	additionalgreeting.Exports.AdditionalGreeting = func() string {
		return "Hello from Go!"
	}
}

func main() {}
```
---

# WAC - A tool for composing WebAssembly Components
- **socket** - A component with an (unsatisified import)
- **plug** - A component with an export that matches the socket
- `wac` composes plug components into socket components to produce a composed component
- simple scenarios "just work"
- a configuration language can handle more complex scenarios

---

# Composition [demo](demo2.html)
- change some go
- build
- change some rust
- build
---

# But I thought you said "Beyond the Browser"

---

# WASI

- WebAssembly System Interface
- Standardized API for WebAssembly to interact with the operating system
- Enables WebAssembly modules to run outside browsers

---

# WASI Evolution

- WASI Preview 1 (2019): First specification with basic system calls
  - modeled after POSIX C api
- WASI Preview 2: Components-based approach
- More expressive interfaces through WIT
- Multiple interface groups:
  - wasi-cli: Command-line functionality
  - wasi-http: HTTP client/server capabilities
  - wasi-io: Input/output operations
  - wasi-filesystem: File system access

---

# WASI Security Model

- Capability-based security: explicit permissions
- no priviliges are the default
- granting permissions is runtime specific

---

# Let's serve our component!
```wit
package demo:hello-wasi-http;

world hello-wasi-http {
  import greet: func(greetee: string) -> string;
  include wasi:http/proxy@0.2.3;
}
```

---
# A peek inside `wasi:http/proxy`
### *very greatly* simplified
```
interface incoming-handler {
  use types.{incoming-request, response-outparam};
  handle: func(request: incoming-request, response-out: response-outparam);
}

world proxy {
  export incoming-handler;
}
```

---
# Implementing in Rust
```rust
use bindings::greet;

impl bindings::exports::wasi::http::incoming_handler::Guest for Component {
    fn handle(_request: IncomingRequest, response_out: ResponseOutparam) {
      let response = OutgoingResponse::new(Fields::new());
      response.set_status_code(200).unwrap();
      let response_body = response.body().unwrap();
      response_body
          .write()
          .unwrap()
          .blocking_write_and_flush(greet("Friends").as_bytes())
          .unwrap();
      OutgoingBody::finish(response_body, None).expect("failed to finish response body");
      ResponseOutparam::set(response_out, Ok(response));
    }
}
```
---

# wasmtime
- runtime which supports the component model
- run - invokes a component
- serve - serves a component over http

---

# Hello server
- change some code
- run it

---

# Use case #1: SAAS plugins
### How do we do it today?
- Webhooks
- API calls
- Customer provided code
  - Lua, javascript, etc
  - DSL or custom language

---

# Problems
- Latency
- Complexity
- Documentation
- Language choice

---

# Solution: WebAssembly Components
- customer provided code
- language agnostic
- securely sandboxed runtime

---

# Example: [WasmCommerce](http://localhost:4000)
### A 100% vibe coded ecommerce platform (Elixir)

---

# Let's add custom shipping calculation
- We want to see the result immediately on the order screen
- Latency is a problem
- webhooks/API calls are not ideal

---

# A shipping calculator WebAssembly component
- create our WIT
- implement our component
- call it from our SAAS
  - requires a runtime (wasmex)

---

# Let's see!

---

# Exercises left for the reader
- Customers providing the component
  - upload or CLI tool
- Fetching and loading component instances per customer

---

# Use case #2: Replacing containers
- Containers are portable, which is great
- They bundle an OS, which is not great
- And also makes em slow to start

---

# Before
![Cloud hosting](cloud_hosting.webp)

---

# After
![Wasm hosting](wasm_hosting.webp)

---

# Hosting for WASM components
- Fermyon Spin
- WasmCloud
- NGINX Unit
- MS Hyperlight WASM

---

# Components on the Edge
- Fastly
- Ferymon/Akamai
- WasmCloud/Akamai
- Edgee

---

# Spin demo
- show toml
- try a demo
- requires tether so we'll see :)

---

# WASI P3
- Async!
- currently we have poll
  - only 1 component can poll at a time
  - everybody is blocked
- future, stream

---

# Questions

![h:500](/images/qrcode.png)
---
