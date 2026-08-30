# qingke-rt

Replaces `ch32v-rt` as the name is not suitable for publishing.

QingKe is the name of the RISC-V core.

This repository preserves the upstream `qingke-rt` history and carries a
small set of runtime changes:

- allow an application to own the `critical-section` implementation;
- preserve the required CH58x global-interrupt enable state;
- initialize the QingKe V3C core used by CH585.

## Usage

```rust
#[qingke_rt::entry]
fn main() -> ! {
    loop {}
}

// Or if you are using the embassy framework
#[embassy_executor::main(entry = "qingke_rt::entry")]
async fn main(spawner: Spawner) -> ! { ... }

#[qingke_rt::interrupt]
fn UART0() {
    // ...
}

// Interrupt provided by the IP core (not peripherals)
#[qingke_rt::interrupt(core)]
fn SysTick() {
    // ...
}

#[qingke_rt::highcode]
fn some_highcode_fn() {
    // ...
    // This fn will be loaded into the highcode(SRAM) section.
    // This is required for BLE, recommended for interrupt handles.
}
```
