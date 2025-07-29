
# Axum vs. Fermyon Spin: A Deep Comparison

## Overview of Axum and Fermyon Spin

**Axum** is a Rust web framework for building HTTP servers. It is built on the Tokio asynchronous runtime and Hyper HTTP library, emphasizing ergonomics and high performance through async/await concurrency. Axum applications compile to a native binary (which can be containerized with Docker) and run as always-on servers listening for requests.

**Fermyon Spin** is a framework and runtime for serverless WebAssembly (WASM) applications. You write your server logic as a WASM component (often in Rust, via the Spin SDK), and the Spin runtime executes these components on-demand in a WASM sandbox. Spin apps are event-driven (e.g. handling HTTP requests) and by default do not run continuously â€“ instead, Spin quickly instantiates a WASM module to handle each incoming request. Spin can be self-hosted or used on Fermyonâ€™s managed cloud, which provides a scaling, on-demand environment for running Spin components.

## Concurrency and Parallelism

**Axum Concurrency:** Axum leverages Tokio to handle many requests concurrently in a single process. By using async functions and Tokioâ€™s multi-threaded executor, an Axum server can process multiple requests in parallel on different OS threads. The framework is â€œasync-first,â€ meaning it fully embraces `async/await` for efficient concurrent I/O. This makes Axum suitable for high-traffic APIs where you manage concurrency via tasks, thread pools, and asynchronous handlers. Because Axum (and Tokio) can spawn tasks, it achieves parallelism on multi-core systems. In practice, an Axum server can utilize all CPU cores and handle many simultaneous connections, making it highly scalable on a single machine.

**Spin Concurrency:** Spin takes a different approach â€“ the developer typically writes single-threaded code, and the platform handles concurrency. In a Spin serverless model, you do not manually spawn threads or tasks for multiple requests; instead, the runtime will create separate WASM component instances to handle incoming requests in parallel. Each HTTP request can be serviced by an isolated instance of your WebAssembly module, which Spin can startup incredibly fast (on the order of microseconds). This means you avoid complex multi-threading in your code â€“ no Mutexes or thread coordination needed â€“ yet the system scales out to handle many requests. For example, Fermyon Cloud can automatically scale a Spin app from zero to thousands of concurrent requests without any explicit concurrency code by the developer. In short, Axum uses multi-threaded concurrency within one process, whereas Spin achieves parallelism by running many short-lived instances of a WASM module (potentially across threads or processes under the hood, managed by the runtime).

## Performance: Docker (Axum) vs. WebAssembly (Spin)

**Axum in a Docker Container:** Running an Axum server in Docker is essentially running a native Linux binary in an isolated environment. Docker imposes negligible overhead on CPU or memory â€“ itâ€™s not an emulator, just uses OS kernel features for isolation. Thus, an Axum service in a container can perform as well as it would on the host. Axum (being compiled to native code) can be extremely fast for request handling and can leverage CPU features directly. For CPU-bound tasks or heavy computations, native Rust code will generally have an edge over WebAssembly.

**Spin on Fermyon (WASM):** Spinâ€™s performance shines in startup latency and density. WebAssembly modules have blazing fast startup times â€“ Spin can instantiate and start handling a request in under a millisecond. This makes Spin ideal for serverless workloads where you might scale down to zero and need to handle bursts of traffic quickly without keeping a process alive. Fermyon reported that on a Mac, a simple Spin app could handle ~28,000 requests per second with an average latency under 0.2 ms, by rapidly spinning up nearly 300,000 isolated WASM instances over 10 seconds. This demonstrates the efficiency of the Spin + Wasmtime runtime for short-lived, I/O-bound tasks.

However, raw execution speed inside a WebAssembly sandbox can be a bit slower than native execution. WebAssembly is designed to be fast, and modern WASM JIT engines compile to machine code, so CPU-heavy code can approach native speeds. But there is overhead in system calls (WASI calls) and at the boundaries between the host and the WASM module. For example, an HTTP Spin component calling out to a database or the filesystem has to go through the WASM runtimeâ€™s interface, which is a bit more overhead than a native call.

**Cold Start vs Warm:** If you already have an Axum server running (warm), each request is handled with almost zero overhead beyond the request processing itself. In Spin, every request might be a â€œcold startâ€ of a new WASM instance â€“ but because that start is so fast, it often doesnâ€™t matter.

**Memory & Density:** WebAssembly modules are typically smaller in memory footprint than a full OS process. Many Spin instances can run in parallel in a single host process (wasmtime can reuse resources with pooling). This means you could pack more Spin instances into the same hardware compared to running many separate containers.

## Pros and Cons of Each Framework

### Axum (Tokio-based Native Server)

**Pros:**
- High performance native execution.
- Rich ecosystem and flexibility.
- Shared state and in-memory caching.
- Maturity and debugging tools.

**Cons:**
- Always-on resource footprint.
- Cold start overhead in container environments.
- Manual concurrency and thread safety required.
- Deployment complexity with Docker/Kubernetes.

### Fermyon Spin (Serverless WebAssembly)

**Pros:**
- Simplified concurrency model.
- Instant scaling and scale-to-zero efficiency.
- Security and sandboxed isolation.
- Portability and polyglot potential.
- Built-in integrations and component composition.

**Cons:**
- WebAssembly execution overhead.
- Stateless by default; persistent state needs external storage.
- Less low-level control and mature tooling.
- Some ecosystem limitations for WASM support.
- Less mature support for long-lived or streaming connections.

## Which is Faster in Practice?

The answer depends on your workload:

- For consistently high traffic: Axum running in a Docker container offers great throughput and low latency.
- For infrequent or spiky traffic: Spin offers better startup speed and automatic scaling.
- For CPU-bound tasks: Axum may perform better due to native execution.
- For maximum density and fast scale-out: Spin performs well due to its lightweight WASM execution model.

## Code Examples: REST API in Axum and Spin

### Axum

```rust
use axum::{extract::State, routing::{get, post}, Json, Router};
use serde::{Deserialize, Serialize};
use std::sync::Mutex;
use std::net::SocketAddr;

#[derive(Serialize, Deserialize, Clone)]
struct Item {
    id: u32,
    name: String,
}

struct AppState {
    items: Mutex<Vec<Item>>,
}

async fn list_items(State(state): State<AppState>) -> Json<Vec<Item>> {
    let items = state.items.lock().unwrap();
    Json(items.clone())
}

async fn add_item(State(state): State<AppState>, Json(new_item): Json<Item>) -> &'static str {
    let mut items = state.items.lock().unwrap();
    items.push(new_item);
    "Item added successfully"
}

#[tokio::main]
async fn main() {
    let state = AppState { items: Mutex::new(Vec::new()) };
    let app = Router::new()
        .route("/items", get(list_items).post(add_item))
        .with_state(state);

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    println!("Axum server listening on {}", addr);
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

### Spin

```rust
use spin_sdk::http::{Request, Response, Method};
use spin_sdk::http_component;
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
struct Item {
    id: u32,
    name: String,
}

#[http_component]
fn handle_items(req: Request) -> Result<Response, Box<dyn std::error::Error>> {
    let method = req.method();
    let path = req.uri().path();

    if method == Method::GET && path == "/items" {
        let sample_items = vec![
            Item { id: 1, name: "Item1".to_string() },
            Item { id: 2, name: "Item2".to_string() },
        ];
        let body_json = serde_json::to_string(&sample_items)?;
        Ok(
            http::Response::builder()
                .status(200)
                .header("content-type", "application/json")
                .body(Some(body_json.into()))?
        )
    } else if method == Method::POST && path == "/items" {
        let body_bytes = req.body().unwrap_or_default();
        if let Ok(new_item) = serde_json::from_slice::<Item>(&body_bytes) {
            println!("Spin: Received new item: {:?}", new_item);
        }
        Ok(http::Response::builder().status(201).body(None)?)
    } else {
        Ok(http::Response::builder().status(404).body(None)?)
    }
}
```

## Conclusion

Both Axum and Fermyon Spin are powerful in their domains. Axum offers complete control and performance for traditional APIs, while Spin provides simplicity and scale for event-driven, serverless applications. Choose based on your workload profile, control needs, and scaling preferences.
