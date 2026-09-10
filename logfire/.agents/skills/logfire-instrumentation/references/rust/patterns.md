# Rust Patterns

## Core Macros

The Rust SDK is built on `tracing` and `opentelemetry`. All `tracing` macros work automatically with Logfire.

### Events (log points)

```rust
logfire::trace!("Detailed trace {detail}", detail = x);
logfire::debug!("Debug info {state}", state = s);
logfire::info!("Normal operation {event}", event = e);
logfire::warn!("Warning {issue}", issue = i);
logfire::error!("Error occurred {err}", err = e);
```

### Spans

```rust
// Scoped - span closes when closure completes
logfire::span!("Processing order {order_id}", order_id = id).in_scope(|| {
    let items = fetch_items(id);
    logfire::info!("Fetched {count} items", count = items.len());
    process_items(items)
});

// Guard-based - span closes when guard is dropped
let _guard = logfire::span!("Long operation {job_id}", job_id = id).entered();
do_work();
// span ends when _guard goes out of scope
```

## Configuration

```rust
use logfire;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let shutdown_guard = logfire::configure()
        .with_install_panic_handler(true)  // captures panics as error spans
        .finish()?
        .shutdown_guard();

    // application code...

    shutdown_guard.shutdown()?;  // flush all pending spans
    Ok(())
}
```

Set `LOGFIRE_TOKEN` in your environment or use the Logfire CLI (`logfire auth`).

## Tracing Crate Compatibility

Any library using `tracing` macros automatically sends data through Logfire:

```rust
use tracing;

tracing::info!("This also appears in Logfire");

#[tracing::instrument]
fn my_function(param: &str) {
    // automatically creates a span with param as an attribute
}
```

## Log Crate Integration

The `log` crate is automatically captured and forwarded to Logfire. Libraries using `log::info!()`, `log::error!()`, etc. will appear in your Logfire dashboard without any additional configuration.

## Async Spans

```rust
use tracing::Instrument;

async fn process_order(order_id: u64) {
    let span = logfire::span!("process order {order_id}", order_id = order_id);
    async {
        fetch_items(order_id).await;
        logfire::info!("Order processed");
    }
    .instrument(span)
    .await;
}
```

## Shutdown

Always call `shutdown()` before program exit to flush pending data:

```rust
// In main()
let shutdown_guard = logfire::configure().finish()?.shutdown_guard();

// ... app runs ...

// Before exit
shutdown_guard.shutdown()?;
```

For web servers using `tokio`, handle shutdown via signal:

```rust
tokio::signal::ctrl_c().await?;
shutdown_guard.shutdown()?;
```
