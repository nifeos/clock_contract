# The stranger cadence

The release's whole claim is that a stranger, given this repository and nothing else, can build
the crate and run its proofs from the README alone. A real agent runs that check on a
cadence, and the weekly `stranger cadence` job fails when the
last run is past 28 days.

What a run is:

1. Clone the repository fresh.
2. Follow the README and nothing else: build, test, and run the proofs.
   The README must name the build and test commands (with any feature a test target needs), the
   bare-metal build with `rustup target add`, the Kani install before the first `cargo kani`, how
   long the proofs take and how to run one harness, what a passing run prints that looks alarming,
   how to run and revert one falsification patch, and how to check the minimum Rust version.
   Each one it lacks is a finding.
3. Note every place the README lied, omitted, or assumed. Those are findings against the README,
   not against the stranger.
4. Record the date on the line below.

Last run: 2026-11-19
- The README names three bare-metal targets but gives no `rustup target add` step; the run machine already had them, so the build succeeded, but a fresh machine would fail with "target may not be installed".
- The README puts the aarch64 and riscv64 targets in a shell comment with no commands of their own; the run wrote `cargo build --lib --target <target>` for each.
- The README says only "`cargo kani`" and gives no way to install Kani and no supported version; Kani was already installed, so the run could not show a newcomer could install it from the README.
- The README does not name the two harnesses or say how to run one alone; the run took the names from the `cargo kani` output and used `cargo kani --harness <name>`.
- The README does not say how to apply, check and revert a falsification patch; the run chose `git apply`, then `cargo kani --harness`, then `git apply -R`.
- The README lists three proven properties after "Two Kani harnesses" without saying which harness proves which.
- The README says the loom run checks three claims, but it printed 8 tests, including restart and initialize cases the README does not mention, and it does not say the loom run replaces the 16 ordinary unit tests.
- The README says "Rust 1.88." but gives no command to check it, and does not say whether the minimum covers the bare-metal, Kani and loom builds; the run guessed `cargo +1.88` and found `rust-version` in Cargo.toml.
- The README heading "EXAMPLES" is in capitals, unlike every other heading.
- The README gives no timings; every step was fast (tests 2.2 s, loom 23 s cold, all proofs 3.8 s).
- Passed as documented: `cargo test` (16 unit tests, 3 doctests), the host build and the three bare-metal `cargo build --lib` builds, `RUSTFLAGS="--cfg loom" cargo test --release` (8 loom tests), `cargo kani` (2 harnesses verified), one harness alone, the policy-bounds falsification failing on its named assertion and reverting clean, and `cargo +1.88 test`.
