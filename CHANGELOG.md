# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- **Four Q formats**, one `@value` type each: `FxQ16_16` (±32768, step
  1.5e-5) is the hub, `FxQ1_15` ([-1, 1)) is the filter and audio
  format whose products do not grow, `FxQ8_24` (±128, step 6.0e-8) is
  the control loop's, and `FxUQ16_16` ([0, 65536)) is the unsigned one
  where subtracting below zero is a refusal rather than a plausible
  negative number.
- **Three families of arithmetic**, chosen per call site: the plain one
  traps on overflow, `sat_` clamps to the end of the range, `wrap_`
  wraps round the container.  A PID integrator wants the second and a
  phase accumulator the third; everything else wants the first, because
  an overflow there is a bug and the trap names the line.
- The multiply's **wide intermediate is free**: two 32-bit raw values
  multiply inside a 64-bit `Int`, which is one `mul` and one `ashr` on a
  Cortex-M4.  That is also why the set stops at 32-bit containers — a
  Q32.32 would need 128 bits, and the language has no type for one.
  `FxUQ16_16.mul` is the one that computes in `u64`, because two
  full-scale unsigned values pass what a signed `Int` holds.
- **micromath's approximations** with their bounds stated and asserted:
  `sqrt` and `invsqrt` to a relative 2e-3, `sin` and `cos` to an
  absolute 3e-3 at every magnitude, `atan` and `atan2` to 5e-3 radians
  with the quadrant exact, `exp` to a relative 3e-3, `log2` and `ln` to
  an absolute 3e-3, `powf` to a relative 1e-2 over a stated domain, and
  `powi` exact.  `floor`, `ceil`, `round`, `trunc` and `abs` are exact
  and are here because the tier refuses `std.math`'s for the same libm
  reason.
- **Decimal in both directions**, exactly: a fixed-point value's
  expansion terminates, so `places` chooses where to stop rather than
  how much to guess.  `format_raw` and `parse_raw` describe the format
  with two integers, which is what lets a caller with a format this
  package does not ship use the text half anyway.

**The device claim is built**, and it is the whole argument for the
package.  `@tier(embedded)` refuses `math.sqrt` because it expands to a
libm call on a soft-float freestanding target, and the compiler's own
error offers fixed-point arithmetic as the first way out.
`tests/embedded_probe.nv` compiles the five arithmetic modules to a
Cortex-M4 ELF for `--target=nrf52-qemu`: a PID step in Q16.16, a filter
tap in Q1.15, a duty ramp in UQ16.16 and a heading out of `atan2`.

**Every fallible direction answers a raw integer**, not a Q number: a
`@value` struct may not be a `Result` payload, so `parse_q16_16` answers
`Result<Int, FxError>` and `fxq16_16.from_raw` is the line that follows.
