# LR2021 Driver

> **This is a fork** of [TheClams/lr2021](https://github.com/TheClams/lr2021).
> `master` is upstream's 0.14.0 with the fixes below on top. Upstream has
> not responded to the open pull requests (#3, #4) for the first of them,
> so they are carried here rather than offered one at a time. The fork is
> not published to crates.io; take it by git revision:
>
> ```toml
> [patch.crates-io]
> lr2021 = { git = "https://github.com/snabb/lr2021.git", rev = "<commit>" }
> ```
>
> Fixes in this fork:
>
> - **No embassy-time features selected on the application's behalf.**
>   Upstream forces `tick-hz-32_768`, `defmt` and `defmt-timestamp-uptime`.
>   The tick rate conflicts with any HAL time driver that selects another
>   rate (embassy-nrf's `time-driver-grtc`, the only one for the nRF54L,
>   selects `tick-hz-1_000_000`), and the build fails with a duplicate
>   `TICK_HZ`. The `defmt` feature now forwards to `embassy-time/defmt`.
> - **NSS is always released**, even when the status returned during a
>   frame reports a failure. That status describes the previous command;
>   returning early left NSS asserted and every later transaction read
>   back as zeros.
> - **The same for `cmd_data_wr` and `cmd_data_rw`**, which skipped the
>   payload and left NSS low on that check.
> - **`cmd_data_wr` writes a payload of any length.** It copied through
>   the 256-byte command buffer and panicked above 256 bytes; FLRC frames
>   go up to 511.
> - **Zeros on MOSI while reading a FIFO or a response**, as Semtech's
>   HAL contract requires. Non-zero bytes can be taken as commands, and
>   the driver was clocking out the caller's buffer.
> - **`get_and_clear_fifo_irq`**, so latched FIFO flags such as an
>   overflow can be cleared as they are read.
> - **`read_intr`**, reading status and `IrqStatus` in one NSS assertion
>   (datasheet §5.2) for polling loops.

[![Crates.io](https://img.shields.io/crates/v/lr2021.svg)](https://crates.io/crates/lr2021)
[![Documentation](https://docs.rs/lr2021/badge.svg)](https://docs.rs/lr2021)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/TheClams/lr2021)

An async, no_std Rust driver for the Semtech LR2021 dual-band transceiver, supporting many different radio protocols including LoRa, BLE, ZigBee, Z-Wave, and more.

## Quick Start

Add this to your `Cargo.toml`:

```toml
[dependencies]
lr2021 = "0.14"
embassy-time = "0.5"
```

Basic usage:

```rust
use lr2021_driver::Lr2021;

let mut radio = Lr2021::new(reset_pin, busy_pin, spi_device, nss_pin);
radio.reset().await?;
// Configure and use your preferred protocol
```

## Hardware Requirements

- Semtech LR20xx transceiver module
- SPI-capable microcontroller
- 3 GPIO pins: Reset (output), Busy (input), NSS/CS (output) (not counting SPI SCK/MISO/MOSI)
- Embassy-compatible async runtime

## Documentation & Examples

- **[API Documentation](https://docs.rs/lr2021-driver)** - Complete API reference
- **[Example Applications](https://github.com/TheClams/lr2021-apps)** - Real-world usage examples on Nucleo boards

## Protocol Test Status

| Protocol | Status | Notes |
|----------|--------|-------|
| LoRa |**Partial** | Basic communication between two LR2021 devices: smallest SF, highest bandwidth. TODO: Ranging |
| BLE | **Partial** | 1MB/s mode, compatible with other BLE devices. TODO: 2Mb/s, Coded |
| FLRC | **Tested** | Basic communication between two LR2021 devices |
| FSK | **Tested** | Generic FSK communication verified |
| Z-Wave | **Tested** | Scan mode tested with ZStick S2, R1-R3 reception |
| OOK | **Partial** | ADSB reception validated, RTS between two LR2021 |
| ZigBee | **Partial** | Reception validated with standard device |
| WiSUN | **Partial** | Basic communication between two LR2021 devices |
| WMBus | **Partial** | Basic communication between two LR2021 devices |
| LR-FHSS | **Unplanned** | TX only (require gateway for test) |
| Sigfox (BPSK) | **Unplanned** | TX only (require gateway for test) |

# LR20xx family
The driver supports the whole LR20xx chip family: LR2012/LR2021/LR2022.
The only difference between each series is chip is the features supported:
 - LR2021 supports all possible features (enabled by default)
 - LR2022 does not support advanced FSK modulation such as FLRC/Zigbee/Zwave
 - LR2012 does not support advanced modulation nor 2.4GHz path

Features in the driver allows to make sure at compile time you are not using supported commands.

## LR2012
When targeting the LR2012 simply disable the default feature:
```toml
[dependencies]
lr2021 = {version = "0.14", default-features = false}
```

## LR2022
When targeting the LR2022, disable the default feature and enable the RF 2.4GHz path:
```toml
[dependencies]
lr2021 = {version = "0.14", default-features = false}

[features]
default = ["lr2021/rf2g4"]
```

