# embassy-qingke-rt

`embassy-qingke-rt` is a small startup and interrupt runtime for WCH QingKe
RISC-V cores. It is a maintained downstream of
[`ch32-rs/qingke`](https://github.com/ch32-rs/qingke) and keeps the original
`qingke-rt` API, linker scripts, interrupt macros, and upstream Git history.

This crate provides reset entry, `.data`/`.bss` initialization, vector-table
setup, QingKe interrupt wrappers, optional SRAM `highcode`, and the
`#[qingke_rt::entry]`/`#[qingke_rt::interrupt]` macros. It is not a PAC, HAL,
clock driver, flash driver, radio stack, or complete per-chip support package.

## Supported core and chip scope

Select exactly one startup-core feature in a chip-facing crate. A core feature
means that the corresponding startup path is available; it does not by itself
guarantee that every MCU using that core has the correct memory map, linker
layout, clocks, or peripheral support.

| Feature | Runtime scope | Known CH58x use |
| --- | --- | --- |
| `v2` | Upstream QingKe V2 startup, including its entry alignment requirements | No CH58x claim |
| `v3` | Upstream generic QingKe V3A-style startup | No automatic CH585 support; use `ch585-v3c` instead |
| `v4` | Upstream QingKe V4 startup with CH58x interrupt-state correction | CH581, CH582, and CH583 family integration; CH582M has downstream hardware use |
| `ch585-v3c` | Dedicated QingKe V3C startup required by CH585 | CH585 and CH585M; downstream startup, GPIO, USB, and BLE integration has exercised this path |

Additional, non-core features:

- `highcode` copies the `.highcode` section from flash to SRAM and points the
  vector table at SRAM. This is commonly required by timing-sensitive CH58x
  interrupt and BLE code.
- `u-mode` retains the upstream user-mode startup option for compatible generic
  cores. CH585 uses its dedicated V3C startup sequence and should not combine
  `u-mode` with `ch585-v3c`.

CH59x and other WCH families are not claimed merely because they may share a
QingKe generation. Add a chip only after its reset CSR sequence, privilege
mode, vector placement, linker layout, and hardware startup have been checked.

Validation boundary:

- `v2`, `v3`, `v4`, and `ch585-v3c` are compile-checked for
  `riscv32imc-unknown-none-elf`;
- CH582M has exercised the `v4` path in downstream firmware;
- CH585M has exercised the `ch585-v3c` path in downstream startup, GPIO, USB,
  and BLE firmware;
- CH581 and CH583 share the family integration target but have not been
  separately hardware-validated by this downstream;
- these observations validate runtime selection, not every peripheral or every
  board using the chip.

## CH58x-specific changes

The downstream changes are deliberately limited to runtime behavior:

1. **Application-owned critical sections.** The runtime no longer enables
   `qingke`'s `critical-section-impl` feature. Reset-time GINTENR and PFIC setup
   is performed directly, allowing the HAL or application to provide the one
   `critical-section` implementation used by the firmware.
2. **CH58x V4 interrupt-state restoration.** The V4 startup path sets both the
   machine interrupt-enable and previous interrupt-enable state with the
   `0x88` GINTENR mask, followed by the required two pipeline NOPs.
3. **CH585 QingKe V3C startup.** `ch585-v3c` programs `CORECFGR` (`0xbc0`) to
   `0x25`, `INTSYSCR` (`0x804`) to `0x03`, CSR `0xbc1` to `0x01`, and
   `mstatus` to `0x88` before entering the application.
4. **No vendor binary dependency.** The runtime is Rust plus the small assembly
   reset path. It does not link a WCH firmware library or BLE binary.

These changes do not replace chip-specific clock, memory, PFIC, USB, radio, or
flash validation in the consuming HAL.

## Dependency examples

Pin a reviewed revision rather than depending on a moving branch.

CH581/CH582/CH583 family integration:

```toml
qingke-rt = {
    git = "https://github.com/ch58-rs/embassy-qingke-rt.git",
    rev = "d2bc2794f0f51246dfa50e424414adb20e0565c9",
    default-features = false,
    features = ["v4", "highcode"],
}
```

CH585/CH585M integration:

```toml
qingke-rt = {
    git = "https://github.com/ch58-rs/embassy-qingke-rt.git",
    rev = "d2bc2794f0f51246dfa50e424414adb20e0565c9",
    default-features = false,
    features = ["ch585-v3c", "highcode"],
}
```

## Runtime usage

```rust
#[qingke_rt::entry]
fn main() -> ! {
    loop {}
}

// Embassy can use the same entry macro.
#[embassy_executor::main(entry = "qingke_rt::entry")]
async fn main(spawner: Spawner) -> ! {
    // Start tasks.
    loop {}
}

#[qingke_rt::interrupt]
fn UART0() {
    // Peripheral interrupt handler.
}

#[qingke_rt::interrupt(core)]
fn SysTick() {
    // QingKe core interrupt handler.
}

#[qingke_rt::highcode]
fn timing_critical_handler() {
    // Copied into the SRAM highcode section when `highcode` is enabled.
}
```

This crate conflicts with `riscv-rt`; a firmware image must use one runtime.

## Upstream source and acknowledgements

This repository was split from the `qingke-rt` crate in
[`ch32-rs/qingke`](https://github.com/ch32-rs/qingke/tree/68a6c7e782e83b9917300b921c11199116447f9b/qingke-rt),
corresponding to the published
[`qingke-rt 0.6.1`](https://crates.io/crates/qingke-rt/0.6.1) source. The
history was filtered by directory rather than imported as a new source dump,
so the original authorship and contributor commits remain available.

Special thanks to **Andelf** for creating and maintaining `qingke` and
`qingke-rt`, to the **ch32-rs team**, and to all upstream contributors whose
runtime, linker, macro, and interrupt work this repository builds upon.

The original project and this downstream are licensed under either the MIT
License or the Apache License 2.0. See `LICENSE-MIT` and `LICENSE-APACHE`.
