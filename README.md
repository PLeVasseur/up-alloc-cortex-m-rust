# up-alloc-cortex-m-rust

Safe fixed startup-memory provider entry point for experimental Cortex-M work.

```rust
use up_alloc_cortex_m_rust::{claim_startup, StartupRegion};
use up_rust_alloc::try_vec_in;

static REGION: StartupRegion<4096> = StartupRegion::new();

// During application initialization:
let allocator = claim_startup(&REGION)?;
let mut values = try_vec_in::<u32, _>(&allocator, 8)?;
let reserved = allocator.seal();
values.push(7); // Within the capacity reserved above.
```

The snippet is a call-shape illustration: errors belong to the application's
setup error type. The complete host example handles them explicitly.

## Cortex profiles

- **Allocation/portability harness:** Cortex-M3, QEMU MPS2 AN385,
  `thumbv7m-none-eabi`. This is an emulated reference, not an automotive MCU.
  The pinned SDK revision admits no execution model for this target, so the
  shared core cannot be built for it; the executed QEMU harness uses Cortex-M7
  on QEMU MPS2 AN500 (`thumbv7em-none-eabihf`) instead.
- **Existing automotive reference:** NXP S32K344 / MR-CANHUBK3, Cortex-M7,
  candidate `thumbv7em-none-eabihf` ABI. Zephyr v4.2.0 is the previously selected
  public enforcement experiment. A supported final ABI/RTOS/toolchain/linker
  binding has not been implemented by this crate.
- **Production evidence direction:** exact S32K3 SAFERTOS port plus its Design
  Assurance Pack, or an incumbent AUTOSAR OS where the project requires it.
  These remain source/evidence inputs; no RTOS certification is inherited by
  this Rust provider.

The `profiles/*.json` files state current evidence and intended target checks.
The `.cargo.toml` files contain only target selection, to pass with `cargo
--config`; they are not linker scripts, board-support packages or admission
receipts. No platform assembly is needed for this fixed-region source.

## Scope and current status

This is an **experimental integration**, not a qualified target allocator or
production SDK release. It uses stable Rust 1.100's standard `Allocator` API in an
optional `no_std + alloc` SDK crate. The production core pin `204eaa24…` and
its Rust 1.88 MSRV are unchanged; the two wires, two uP-L1 transports and both
providers share the experimental SDK revision named below. Ferrocene
target/toolchain and `alloc` qualification are separate evidence questions; no
such acceptance is claimed.

The manifest pins [the public experimental SDK](https://github.com/PLeVasseur/up-rust/tree/08c0a83e174afea6b227e74c9dfaadf8b0828258)
to `08c0a83e174afea6b227e74c9dfaadf8b0828258` (branch `codex/allocator-mcu-spike`). Both provider repositories use that same revision. There are no
private SDK copies, local path dependencies, or patch overrides in this package.
This SDK revision includes the shared native32 execution-model selection described below.

The implementation belongs to the shared SDK's `up-rust-alloc::fixed`, **once**.
This independent crate supplies a safe public entry point and platform-profile
configuration; it does not copy an arena, modify private SDK modules or register
as a privileged provider. Other authors can implement Rust's standard unsafe
`Allocator` in their own crate or use these same public SDK hooks.

`up-rust-core` remains `no_std`, `no_alloc`, dependency-free and feature-free.
The alloc helper does not depend back on either provider. Native payload loans
and retained receive samples remain owned by the underlying transport; this
startup allocator is for SDK/application memory, not a native payload copy layer.

## What fixed-region allocation means

The caller provides `StartupRegion<N>`, whose storage is claimed once through the
SDK. All reservations use core's checked alignment/capacity rules. The common
allocator is local (`!Sync`); it has no RTOS lock, interrupt handler or hidden
cross-thread promise. An application may use distinct regions in distinct
contexts. This optional provider does not change any shared SDK transport handle
contract.

`seal()` permanently prevents new allocator reservations while preserving live
owners and pre-reserved capacity. This is **not** the SDK's full setup/runtime
activation and does not make arbitrary `Vec`/`Box` methods or element destructors
native-runtime safe. Use fallible reservation during setup; ordinary infallible
operations can still panic or reach allocation-error handling. Empty/ZST standard
collections may succeed without consulting the allocator even after sealing.

Deallocation retains physical backing; growth/shrink may consume additional
permanent region bytes. Dropping or forgetting an owner cannot reopen its region.
Core `MemoryReport` counts linked logical reservations, not a full memory budget.
Automatic fixed-memory sizing remains unfinished SDK work.

## Builds and final links

The included `examples/fixed_region.rs` is a **host** application example using
safe public APIs. Its `expect`/asserts are host demonstration code, not approved
runtime failure handling. No bare-metal boot image or RTOS integration is hidden
inside that example. The smallest host check is:

```sh
cargo +1.100.0 check --locked --lib --examples
cargo +1.100.0 run --locked --example fixed_region
```

A `no_std + alloc` library check alone does not establish a final MCU image.
A preserved Rust 1.100 staticlib probe requires global-allocator registration even
when selected operations use only an explicit allocator. Registration, allocation
error handling, panic handling, board memory layout and startup remain final-link
work. No global heap fallback is introduced here. A rejecting global registration
lives in the separate QEMU test harness, which is not part of this provider or a
production runtime guarantee. After its independent review, that harness linked
Rust 1.100 release images from this provider's current library source and a
frozen pre-publication SDK source tree (native32/allocator sources before the
SDK's later G2/G4 composition). On November 14, 2026 the Cortex-M7 image passed
its native32 and allocator probes once under QEMU `mps2-an500`, exit 0 (ELF
SHA-256 `36a2364f…`). That run binds that exact image only, not this provider at
the pinned SDK revision.

At this SDK revision the shared core compiles for a bare-metal target only with
an explicitly selected SDK-owned execution model. For `thumbv7em-none-eabihf`
that is `RUSTFLAGS='--cfg up_rust_native32_model="cortex-m7-an500"'`, the
SDK's QEMU MPS2 AN500 Cortex-M7 image profile; it is not an S32K344 board
binding. The SDK build script rejects the AN385 `thumbv7m-none-eabi` harness at
this revision because no execution model exists for it. No private memory-only SDK fork, force-enabled atomics or
silently disabled core modules are used. Source, host compilation/execution,
target compilation, final link, emulator execution and hardware qualification
must be recorded separately. G1/G2/G4 and clock/entropy/exclusion are not solved
by this allocation experiment.

At this SDK revision, locked Rust1.100.0 checks passed for the host library, the
host example and the `thumbv7em-none-eabihf` library with the `cortex-m7-an500` model, on an
aarch64 macOS host. See [VALIDATION.md](VALIDATION.md) for scope and
[SDK-PIN.json](SDK-PIN.json) for the dependency pin. Final link, QEMU and hardware
results remain open for this provider revision.
