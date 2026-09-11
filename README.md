# fixedpoint-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

Arithmetic for a core that has no floating-point unit, in two halves.

**Q-format fixed point** — four `@value` types, each one integer with a
binary point at a fixed place: add, subtract, multiply with the wide
intermediate, divide, in three families (trapping, saturating,
wrapping), with conversion to and from `Int` and `Float`, rounding,
comparison, and decimal in both directions.

**micromath's float approximations** — `sqrt`, `invsqrt`, `sin`, `cos`,
`tan`, `atan`, `atan2`, `exp`, `ln`, `log2`, `powf` and `powi`, each
with a stated error bound that `tests/fxmath_tests.nv` asserts, written
in arithmetic a soft-float core can afford.

## Why it exists: the compiler says so

A function that carries `@tier(embedded)` and calls `math.sqrt` is
refused, and the error names the reason:

```
tier error: function 'root' is @tier(embedded) but calls 'math.sqrt' —
it lowers to an `llvm.sqrt`-family float intrinsic, which on a
soft-float freestanding target LLVM expands to a libm call, and linking
libm into a 64 KB image is a hidden cost this tier refuses (the IR looks
free; the object file is not).  Use fixed-point arithmetic, or link your
own implementation through @ffi and call that.  The allocation-free half
of `math` (min, max, is_nan, the constants) IS admitted [E4000]
```

`sqrt`, `pow`, `log`, `log2`, `log10`, `exp`, `sin`, `cos`, `tan`,
`asin`, `acos`, `atan`, `atan2`, `hypot`, `floor`, `ceil`, `round` and
`trunc` are all in that set.  The error offers three ways out, and this
package is two of them: the fixed-point arithmetic it names first, and a
set of approximations that are not an `@ffi` call into somebody's libm.

## The shipped Q formats, and why these four

A type cannot carry an integer parameter — there are no const generics,
and a `@value` struct takes no type parameter at all — so each format is
a type written out by hand rather than a `Fixed<16, 16>`.  That is the
cost.  What it buys is the thing the `fixed` crate is really for:
**adding a Q16.16 to a Q8.24 is a compile error**, not a silent scaling
bug, and the two conversions between them are calls a reader can see.

| type | range | step | reach for it when |
| --- | --- | --- | --- |
| `FxQ16_16` | ±32768 | 1.5e-5 | nothing says otherwise — the hub, and where the worked examples are |
| `FxQ1_15` | [-1, 1) | 3.1e-5 | a filter, an audio sample, a normalised axis: the product of two in-range numbers stays in range |
| `FxQ8_24` | ±128 | 6.0e-8 | a control loop whose error term is small and whose gain is large |
| `FxUQ16_16` | [0, 65536) | 1.5e-5 | a duty cycle, an elapsed time, a distance: no sign, double the integer range, and subtracting below zero is a refusal |

**Why the set stops here.** The multiply needs the product of two raw
values before it shifts the scale back out — 64 bits for a 32-bit
format — and `Int` IS 64 bits, so that intermediate is free: one `mul`
and one `ashr` on a Cortex-M4.  A Q32.32 would need a 128-bit
intermediate and the language has no type for one.  `FxQ8_24.div`
already uses fifty-five of the sixty-four bits when it shifts, and a
Q4.28 would not fit.  So the containers stop at 32 bits, and that is
arithmetic rather than taste.

`FxUQ16_16` is the one exception that needed care: two full-scale
unsigned raw values multiply past what a *signed* `Int` holds, so that
one multiply computes its intermediate in `u64`.

## The layer, and why

`core`.  Integer and float arithmetic over values the caller already
holds; no function declares an effect, nothing is read and nothing is
written.

It carries `tests/embedded_probe.nv`, so the device claim is **built**
rather than asserted: the five arithmetic modules speak `Int`, `Float`,
`u64` and nothing else, and the probe compiles to a Cortex-M4 ELF for
`--target=nrf52-qemu` — a PID step in Q16.16, a filter tap in Q1.15, a
duty ramp in UQ16.16, and a heading out of `atan2`.  That claim is the
whole argument for the package: everything here is `+`, `-`, `*`, `/`,
comparisons and the two casts between `Int` and `Float`, which is
exactly the set the tier admits.

`fixedpoint` is deliberately outside the probe: it speaks `Str`, and one
host-only function anywhere in a compilation unit is an undefined symbol
at embedded link time whether or not the firmware calls it.

**A device that must print one of these numbers** calls
`fixedpoint.format_raw`'s arithmetic itself and hands the integer and
fractional parts to [`numfmt-nv`](https://github.com/novolang/numfmt-nv),
which writes digits into a buffer the caller owns and allocates nothing.
That is one line a firmware can afford; a `Str` is not.  This package
does not depend on numfmt-nv, because printing is the caller's step and
because the embedded probe is built from the package's own core modules
and nothing they depend on.

## Adding it, and checking it

```bash
novo pkg add fixedpoint-nv   # into your novo.toml
novo pkg build               # type- and effect-check the package
novo test --isolate tests/fixedpoint_tests.nv
novo test --isolate tests/fxmath_tests.nv
```

Both suites are red today and that is the point of the release: every
assertion fails with `not implemented: <module>.<fn>`.  They turn green
one at a time as bodies land.

**One caveat on reading that output.** Three assertions in
`fixedpoint_tests.nv` use `test.assert_raises` to say that an operation
traps, and `todo()` panics, so `assert_raises` is satisfied by the stub
and those three are GREEN today while meaning nothing.  That is a
toolchain defect rather than a property of this package; the tests are
written the way they should be written once the bodies land.

## The one example that will work

```novo
use fxq16_16
use fixedpoint

fn main() [io]
    // A gain given as a ratio, with no float on the device.
    let kp = fxq16_16.from_ratio(3, 500)
    let err = fxq16_16.sub(fxq16_16.from_int(20), fxq16_16.from_raw(1376256))

    // The integrator saturates, so an actuator at full travel stays
    // there rather than wrapping to the far stop.
    let out = fxq16_16.sat_add(fxq16_16.zero(), fxq16_16.mul(kp, err))

    println(fixedpoint.format_q16_16(out, 4))
```

## The load-bearing interface

`FxQ16_16`, and the three families of arithmetic over it:

```novo
pub @value
struct FxQ16_16
    raw: Int
```

One integer in the caller's frame.  `raw` is the number times 65536,
held inside the 32-bit container, and the invariant that it never leaves
that container is what lets a caller read the field and hand it to a
peripheral.

Everything else follows from two decisions:

- **The step is constant.** A float's step near 1000 is a thousand times
  its step near 1; a Q16.16's is 2^-16 everywhere. That is why a
  fixed-point control loop converges where a float one oscillates, and
  why `epsilon()` is a number a termination test can compare against
  rather than a scale-dependent guess.
- **Overflow is a choice the caller makes, per call site.** `add` traps
  (SPEC § 13.2), `sat_add` clamps, `wrap_add` wraps. A PID integrator
  wants the second, a phase accumulator wants the third, and everything
  else wants the first — because an overflow there is a bug and a trap
  names the line. Shipping only one of the three would have made two of
  those three consumers write it themselves.

Three consequences a reviewer should push on:

- **A parse answers a raw, not a number.** A `@value` struct may not be
  the payload of a `Result` — it is unboxed and the position has no
  unboxed lowering — so `fixedpoint.parse_q16_16` answers
  `Result<Int, FxError>` and `fxq16_16.from_raw` is the line that
  follows. Every fallible direction in the package has that shape.
- **The text engine takes two integers, not a type.**
  `format_raw(raw, frac_bits, places)` and
  `parse_raw(s, frac_bits, total_bits, signed)` describe the format with
  numbers, and the four typed wrappers are one line each on top. That is
  the closest thing to the type parameter the language does not have,
  and it is what lets a caller who needs a Q12.20 use the text half
  anyway.
- **The bounds in `fxmath` are a contract.** Each function's doc comment
  states an error bound over a stated domain and `tests/fxmath_tests.nv`
  asserts that number. Widening one is a breaking change; tightening one
  is not.

## The reference implementations

`fixed` (Rust, MIT/Apache-2.0) for the Q-format surface and the
semantics of the three families, and `micromath` (Rust, Apache-2.0) for
the approximations and the algorithms behind them — the reciprocal-root
estimate with one Newton step, the parabolic sine with a correction
pass, the rational arctangent with the reciprocal identity outside
`[-1, 1]`, and the exponent-field logarithm.

Both crates generate per-type code with macros. There is no macro here,
so what they generate this package writes out: four modules that are the
same thirty-odd functions at four different scales, with each header
saying what DIFFERS and `fxq16_16` carrying the semantics once. The
repetition is the price of `FxQ16_16 + FxQ8_24` being a compile error.

The error bounds in the doc comments are the ones this package asserts
for its own implementations over the domains it states; they follow
micromath's algorithms, and the numbers are in
`tests/fxmath_tests.nv` rather than only in prose.

## Status

| function | implemented |
| --- | --- |
| `fxq16_16` — 31 functions over `FxQ16_16` | no |
| `fxq1_15` — 33 functions over `FxQ1_15`, with the two hub conversions | no |
| `fxq8_24` — 33 functions over `FxQ8_24`, with the two hub conversions | no |
| `fxuq16_16` — 31 functions over `FxUQ16_16`, with the two hub conversions | no |
| `fxmath.sqrt`, `.invsqrt` | no |
| `fxmath.sin`, `.cos`, `.tan`, `.atan`, `.atan2` | no |
| `fxmath.exp`, `.ln`, `.log2` | no |
| `fxmath.powf`, `.powi` | no |
| `fxmath.floor`, `.ceil`, `.round`, `.trunc`, `.abs` | no |
| `fixedpoint.format_raw`, `.parse_raw`, `.format_name` | no |
| `fixedpoint.format_q16_16` … `.parse_uq16_16` (four pairs) | no |
| `fixedpoint.FxError.message` | no |
