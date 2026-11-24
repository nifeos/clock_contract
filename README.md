# clock_contract

[![check](https://github.com/partsfoundry/clock_contract/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/partsfoundry/clock_contract/actions/workflows/ci.yml?query=branch%3Amain)
[![proofs](https://github.com/partsfoundry/clock_contract/actions/workflows/proofs.yml/badge.svg?branch=main)](https://github.com/partsfoundry/clock_contract/actions/workflows/proofs.yml?query=branch%3Amain)
[![license](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue.svg)](https://github.com/partsfoundry/clock_contract#license)
[![minimum Rust version](https://img.shields.io/badge/rust-1.88%2B-blue.svg)](https://github.com/partsfoundry/clock_contract/blob/main/Cargo.toml)

The wall clock as a contract between a clock service and everything that reads or sets the time.
It holds the layout of a shared clock page, the small protocol for proposing a new time, and the
policy a proposal must meet. `no_std`, no dependencies outside an optional model-checking build. A clock
service, its readers and a standard library's time support share this one definition, so they
cannot drift. On its own the crate maps nothing and sets nothing; it is useful with a clock service
that maps the page and a loader that grants it.

Reading the clock is a read-only mapping of the page. Setting it is a read/write mapping of the same
page. Proposing a time is a message to an endpoint, which the service answers by applying the policy
here. Wall-clock time is the monotonic counter plus an offset, and only the offset is ever written,
so stepping the clock cannot disturb a monotonic `Instant`. Times are nanoseconds since the Unix
epoch in a `u64`, which runs out in 2554.

## The contract

- The page: seven 64-bit words (`WORDS`), the rest of the frame reserved and zero. Word 0 is
  `MAGIC` (ASCII `CLOCKv01`), word 1 the publish count, word 2 the writers' claim, words 3 and 4 the
  state and the offset in nanoseconds of copy 0, and words 5 and 6 the same for copy 1. The publish
  count's low bit names the copy readers use. A frame without the magic reads as the unknown state.
- The identity does not change: the magic `CLOCKv01`, `VERSION` 1, `INTERFACE_ID` 8 and the
  seven-word layout above. A layout break ships as a new magic and version together, never a new id. `VERSION` (1) is the digits of the magic: a
  reader built for one version reads another's page as no clock at all. A layout break bumps both.
- States (`state`): `UNKNOWN`, `REAL_TIME_CLOCK`, `SET`, `SYNCHRONIZED`. `UNKNOWN` is a real answer. A machine with no
  usable clock must say so; reporting 1970 plus uptime is the failure this contract exists to
  prevent.
- `ClockPage` is the page as seen through one mapping. There are many readers, several writers and
  no lock can cross an address space, so a reader never waits on a writer. `publish` takes the claim
  word with a compare-exchange, so writers exclude each other, fills the copy the count does not
  name, advances the count and gives the claim back. `read` returns one consistent `Reading` (state,
  offset, generation, where generation is the publish count), and retries only when a publish
  finished during its own read, which a stopped writer never does. `new` is `unsafe`: the address
  must be a mapped, 8-byte-aligned frame that stays mapped.
- The conversion: `wall_nanoseconds(offset, monotonic)` saturates rather than wraps, and `offset_for` is its
  inverse.
- Proposals (`propose`, `status`): a request is `request(PROPOSE)` with the proposed time as the second
  word, or `request(STATE)` to ask what the clock knows. The opcode sits in bits 63:56. The reply's first
  word is a `status` (`ACCEPTED`, `REFUSED_IMPLAUSIBLE`, `REFUSED_TOO_FAR_FORWARD`,
  `REFUSED_TOO_FAR_BACKWARD`, `BAD_REQUEST`) and its second the wall-clock nanoseconds in force.
- The policy (`policy::decide`) refuses as implausible a proposal outside the window from 2026-01-01 (`NOT_BEFORE_NANOSECONDS`) to
  2100-01-01 (`NOT_AFTER_NANOSECONDS`), and the window also judges what a hardware
  clock reports. Once the clock knows the time, one accepted proposal may move it forward by at most
  an hour (`MAXIMUM_STEP_FORWARD_NANOSECONDS`) and backward by at most a second (`MAXIMUM_STEP_BACKWARD_NANOSECONDS`).
  Moving backward makes instants happen twice, which breaks log ordering and expiry. A clock that
  is `UNKNOWN` accepts any plausible proposal. The window is a floor on plausibility, not a claim of
  accuracy.
- `real_time_clock` names the kinds of hardware clock a service may read: `NONE`, `PL031`, `GOLDFISH`
  and `CMOS`. `PL031` and `GOLDFISH` name a register layout the service reads through a mapped
  page; `CMOS` names the two I/O ports, `INDEX_PORT` and `DATA_PORT`, it reads through a
  port capability.

## EXAMPLES

```rust
use clock_contract::{NANOSECONDS_PER_SECOND, offset_for, policy, state, status, wall_nanoseconds};

// The machine has been up for 5 seconds and the service learns the wall time.
let monotonic = 5 * NANOSECONDS_PER_SECOND;
let now = 1_800_000_000 * NANOSECONDS_PER_SECOND;
assert_eq!(policy::decide(state::UNKNOWN, 0, now), status::ACCEPTED);
let offset = offset_for(now, monotonic);
assert_eq!(wall_nanoseconds(offset, monotonic), now);

// A proposal a day ahead of a synchronized clock is refused.
assert_eq!(
    policy::decide(state::SYNCHRONIZED, now, now + 86_400 * NANOSECONDS_PER_SECOND),
    status::REFUSED_TOO_FAR_FORWARD,
);
```

## What is proven

Two Kani harnesses cover every input:

- `verification::a_proposal_is_accepted_only_inside_the_policy_bounds`: `policy::decide` accepts a
  proposal only inside the plausibility window and the step bounds, and accepts every plausible
  proposal when the time is unknown.
- `verification::the_request_word_and_the_offset_round_trip`: the request word's opcode and the
  offset arithmetic round trip.

Each has a patch under `falsifications/` that makes it fail.

The loom run checks the clock page against the C11 model, not against the Arm or RISC-V memory
models. Its models hold three claims. A reader never returns a torn pair, including a reader
that sleeps across two publishes and one that reads while the page is initialized. A writer stopped
at any point inside a publish never makes a reader wait, and a restart that clears its claim keeps
the last publish. Two writers never interleave. Weakening any ordering on the page fails one of
them, except the release that gives the claim back, which loom passes and the argument in the
source carries.

## Building and testing

The crate needs 64-bit atomics (`target_has_atomic = "64"`): every word of the page is an
`AtomicU64`, so a target without them fails to build with a message that says so.

```sh
cargo test
rustup target add x86_64-unknown-none aarch64-unknown-none-softfloat riscv64imac-unknown-none-elf
cargo build --lib --target x86_64-unknown-none
cargo build --lib --target aarch64-unknown-none-softfloat
cargo build --lib --target riscv64imac-unknown-none-elf
```

Run `rustup target add` once, before the first bare-metal build. `cargo test` runs the unit tests
and the doctests in a few seconds.

### Running the loom models

```sh
RUSTFLAGS="--cfg loom" cargo test --release
```

This runs the loom models in place of the unit tests, so it reports fewer tests than `cargo test`,
each a model rather than a unit test. It takes under half a minute, most of it compiling loom.

### Running the proofs

Install Kani 0.68.0, the version CI uses, with `cargo install --locked kani-verifier --version
0.68.0`, then `cargo kani setup`, which downloads a model checker. Plain `cargo kani` runs both
harnesses in a few seconds and ends `Complete - 2 successfully verified harnesses, 0 failures, 2
total.` To run one:

```sh
cargo kani --harness verification::a_proposal_is_accepted_only_inside_the_policy_bounds --exact
```

### Running one falsification patch

```sh
git apply falsifications/verification.a_proposal_is_accepted_only_inside_the_policy_bounds.patch
cargo kani --harness verification::a_proposal_is_accepted_only_inside_the_policy_bounds --exact   # ends VERIFICATION:- FAILED
git checkout -- src
```

The harness name is the part of the patch's file name after `verification.`, and each patch turns
its own harness red.

## BUGS

- A writer stopped while it holds the claim blocks every later writer until something clears the
  claim. Readers keep reading the last publish. Nothing here clears a claim it did not take.

## Minimum Rust version

Rust 1.88, for the host and the bare-metal builds alike. To check it:

```sh
rustup toolchain install 1.88 --profile minimal --target x86_64-unknown-none
cargo +1.88 test
cargo +1.88 build --lib --target x86_64-unknown-none
```

## License

Licensed under either of MIT (LICENSE-MIT) or Apache-2.0 (LICENSE-APACHE), at your option.
