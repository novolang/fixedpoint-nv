# fixedpoint-nv

**Fixed-point arithmetic** represents a fractional number as an ordinary
integer with the binary point at a fixed place. A Q16.16 number is the value
multiplied by 65536 and stored in 32 bits: sixteen bits of integer part,
sixteen of fraction. This package brings four such formats to novo-lang,
together with a set of approximations for square roots, logarithms and
trigonometry. It exists because a microcontroller with no floating-point unit
cannot afford the library those functions normally come from. The reference
implementations are the Rust crates [fixed](https://docs.rs/fixed) and
[micromath](https://docs.rs/micromath).
[pid-nv](https://novo-lang.org/packages/pid-nv) is built on it.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it
is implemented. Version 0.1.0 will be the first working release.

## What it is

A **Q format** is written `Qm.n`: `m` bits of integer part and `n` bits of
fraction, in one signed container. The value stored is the real number times
two to the power `n`, rounded to an integer. Adding two Q16.16 numbers is one
integer addition. Multiplying them is one integer multiplication followed by
a shift, because the product has twice as many fractional bits as either
operand.

Two things follow that matter more than the speed.

The **step is constant**. A floating-point number's step near 1000 is a
thousand times its step near 1. A Q16.16 number's step is two to the minus
sixteenth everywhere in its range. A control loop written in fixed point
therefore converges where the same loop in floating point oscillates around a
setpoint far from zero.

The **range is fixed and small**, so arithmetic can leave it. What happens
then is a choice this package makes the caller state at each call site. The
plain operations **trap**, which stops the program at the line. The
saturating operations **clamp** to the end of the range. The wrapping
operations **wrap** round the container.

These are the four formats.

| Type | Range | Step | Reach for it when |
| --- | --- | --- | --- |
| `FxQ16_16` | ±32768 | 1.5e-5 | nothing says otherwise. It is the format the others convert through. |
| `FxQ1_15` | -1 to just under 1 | 3.1e-5 | a filter tap, an audio sample, a normalised axis. The product of two in-range values stays in range. |
| `FxQ8_24` | ±128 | 6.0e-8 | a control loop whose error is small and whose gain is large. |
| `FxUQ16_16` | 0 to just under 65536 | 1.5e-5 | a duty cycle, an elapsed time, a distance. No sign, twice the integer range, and subtracting below zero is a refusal. |

The second half of the package is a set of **approximations**: functions that
answer a value close enough to the true one, computed in arithmetic a device
can afford. Each carries an **error bound**, which is how far the answer may
be from the true value, over a stated **domain**, which is the range of
inputs the bound holds for.

| Function | Bound | Domain |
| --- | --- | --- |
| `sqrt`, `invsqrt` | relative 2e-3 | x above 0 |
| `sin`, `cos` | absolute 3e-3 | every finite x |
| `tan` | absolute 3e-3 | every finite x |
| `atan`, `atan2` | absolute 5e-3 radians, quadrant exact | every finite x |
| `exp` | relative 3e-3 | -10 to 10 |
| `ln`, `log2` | absolute 3e-3 | x above 0 |
| `powf` | relative 1e-2 | x from 1e-3 to 1e3, y within 4 |
| `powi` | exact | binary exponentiation |
| `floor`, `ceil`, `round`, `trunc`, `abs` | exact | every finite x |

## Install

```
novo pkg add fixedpoint-nv
```

## Example

```novo
use fxq16_16
use fixedpoint

fn main() [io]
    // A gain of 0.006, written as a ratio because the device has no float.
    let kp = fxq16_16.from_ratio(3, 500)

    // The error term: a setpoint of 20 less a measurement held as a raw
    // value, which is the number times 65536.
    let err = fxq16_16.sub(fxq16_16.from_int(20), fxq16_16.from_raw(1376256))

    // The saturating add clamps at the format's limit. An integrator that
    // wrapped instead would drive an actuator to the far stop.
    let out = fxq16_16.sat_add(fxq16_16.zero(), fxq16_16.mul(kp, err))

    // Four decimal places. This line is host-only: it makes a string.
    println(fixedpoint.format_q16_16(out, 4))
```

Build and test with `novo pkg build` and
`novo test --isolate tests/fixedpoint_tests.nv`. Today `novo test` fails on
purpose: every test reaches a `not implemented: <module>.<fn>` panic. The
tests are the specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `fxq16_16` | The Q16.16 type and thirty-one functions over it: the three arithmetic families, the conversions, rounding, comparison and the constants. This is where the semantics are written out. |
| `fxq1_15` | The same surface at the Q1.15 scale, plus the two conversions to and from Q16.16. |
| `fxq8_24` | The same surface at the Q8.24 scale, plus the two conversions to and from Q16.16. |
| `fxuq16_16` | The same surface unsigned, plus the two conversions to and from Q16.16. |
| `fxmath` | The approximations over `Float`, each with its error bound, and the five exact rounding functions. |
| `fixedpoint` | Decimal in both directions, as text, and the error type. Host only: it makes strings. |

## How to choose an entry point

**Start with `fxq16_16`.** It is the format the worked examples use and the
one the other three convert through. Move to another only when its range or
its step is wrong for the quantity.

**Use `fxmath` when a value is already a `Float` and the function is not
available.** Every function there is written in additions, multiplications
and comparisons, so it compiles for a device where the standard library's
version does not.

**Use the untyped text functions for a format this package does not ship.**
`fixedpoint.format_raw` and `parse_raw` describe a format with plain numbers
rather than a type, so a Q12.20 value can be printed and read with them. The
four typed pairs are one line each on top.

## The rules a user needs

1. **Choose the overflow behaviour at the call site.** `add` traps
   (SPEC section 13.2), `sat_add` clamps, `wrap_add` wraps. A control loop's
   integrator wants the saturating family. A phase accumulator wants the
   wrapping one. Everything else wants the trapping one, because an overflow
   there is a bug and a trap names the line.
2. **Adding two different formats is a compile error.** `FxQ16_16` and
   `FxQ8_24` are different types, and the conversion between them is a call a
   reader can see. That is the point of writing four types out rather than
   one parameterised one.
3. **`raw` is the number times two to the power of the format's fractional
   bits, and it never leaves its container.** A caller may read the field and
   hand it straight to a peripheral.
4. **A fallible direction answers a raw integer, not a formatted value.** A
   `@value` struct may not be the payload of a `Result`, so
   `fixedpoint.parse_q16_16` answers a `Result` over `Int` and
   `fxq16_16.from_raw` is the line that follows it.
5. **The decimal expansion terminates, so `places` chooses where to stop.** A
   fixed-point value's decimal form is exact and finite. Printing is not an
   approximation here, unlike printing a float.
6. **The error bounds are a contract.** Each function in `fxmath` states its
   bound over its domain, and `tests/fxmath_tests.nv` asserts that number.
   Widening a bound is a breaking change. Tightening one is not.
7. **`tan`'s bound is absolute, not relative.** It is `sin` divided by `cos`,
   so the two bounds compound and the relative error grows without limit as
   the cosine approaches zero.
8. **`ln`'s bound is absolute, not relative.** The logarithm of 1 is zero, and
   no relative bound around zero is meaningful.
9. **`exp` outside its stated domain is still monotone and still the right
   order of magnitude**, but the bound does not hold there.
10. **Subtracting below zero in the unsigned format is a refusal, not a large
    positive number.** That is what `FxUQ16_16` is for.
11. **The unsigned multiply computes its intermediate as an unsigned 64-bit
    value.** Two full-scale unsigned raw values multiply past what a signed
    `Int` holds. Every other multiply in the package fits in a signed `Int`.

## Running on a microcontroller

The package states that its modules run on a device with no heap allocator,
and the compiler checks that claim on every build. It covers the five
arithmetic modules, which speak `Int`, `Float`, `u64` and nothing else.

This claim is the reason the package exists. A function at the device tier
that calls `math.sqrt` is refused, because that call lowers to a float
intrinsic that a soft-float freestanding target expands into a library call,
and linking that library into a 64 KiB image is a cost the tier will not pay.
The refusal names three ways out, and this package is two of them:
fixed-point arithmetic, and approximations that are not a foreign call into
somebody else's library. `sqrt`, `pow`, `log`, `log2`, `log10`, `exp`, `sin`,
`cos`, `tan`, `asin`, `acos`, `atan`, `atan2`, `hypot`, `floor`, `ceil`,
`round` and `trunc` are all in the refused set.

`tests/embedded_probe.nv` is the claim as a program that either builds or
does not. It runs a control-loop step in Q16.16, a filter tap in Q1.15, a
duty ramp in UQ16.16 and a heading out of `atan2`.

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

That command was run against this release. It produces a Cortex-M4
executable, `embedded_probe.elf`. The probe builds; it is not run, because
every function it calls is a `todo()` that would panic on the first line.

The `fixedpoint` module is outside the claim. It makes strings, and one
host-only function anywhere in a compilation unit is an undefined symbol at
link time on a device, whether or not the firmware calls it. A device that
must print one of these numbers does the arithmetic of
`fixedpoint.format_raw` itself and hands the integer and fractional parts to
[numfmt-nv](https://novo-lang.org/packages/numfmt-nv), which writes digits
into a buffer the caller owns and allocates nothing.

## What is not included

- **A format parameterised by its bit positions.** novo-lang has no integer
  type parameters, and a `@value` struct takes no type parameter at all.
  Each format is written out by hand.
- **Containers wider than 32 bits.** A multiply needs the product of two raw
  values before it shifts the scale back, which is 64 bits for a 32-bit
  format, and `Int` is 64 bits, so that intermediate is free. A Q32.32 would
  need a 128-bit intermediate and the language has no type for one.
  `FxQ8_24`'s divide already uses fifty-five of the sixty-four bits.
- **Formats between the four shipped.** `fixedpoint.format_raw` and
  `parse_raw` work for any of them. The typed arithmetic does not.
- **Printing on a device.** See "Running on a microcontroller".
- **A dependency on numfmt-nv.** Printing is the caller's step, and the
  device probe is built from this package's own modules and nothing they
  depend on.
- **Exact transcendental functions.** Every function in `fxmath` is an
  approximation with a stated bound. A program that needs more digits needs a
  machine with a floating-point unit and the standard library's `math`.

## Related packages

- [pid-nv](https://novo-lang.org/packages/pid-nv) is a proportional-integral-
  derivative controller. Its fixed-point form is written over `FxQ16_16`.
- [numfmt-nv](https://novo-lang.org/packages/numfmt-nv) writes digits into a
  buffer the caller owns. It is how a device prints one of these numbers.
- [units-nv](https://novo-lang.org/packages/units-nv) is physical quantities
  and conversions between them, on a host. It answers what a number means;
  this package answers how a number is stored.
- `std.math` in the standard library is the real thing, on a machine that can
  afford it: correctly rounded, and refused at the device tier for the reason
  above.
- [bitfield-nv](https://novo-lang.org/packages/bitfield-nv) reads a sensor's
  raw register field. A value read with it is usually converted to one of
  these formats next.

## Tests

```bash
novo test --isolate tests/fixedpoint_tests.nv   # 45 tests: the four formats
novo test --isolate tests/fxmath_tests.nv       # 19 tests: the bounds
```

The references are `fixed` for the Q-format surface and the meaning of the
three families, and `micromath` for the approximations: the reciprocal-root
estimate with one Newton step, the parabolic sine with a correction pass, the
rational arctangent with the reciprocal identity outside -1 to 1, and the
logarithm read out of the exponent field. The bounds asserted here are this
package's own, over the domains stated above.

The tests compile today and fail at run, each on the
`not implemented: <module>.<fn>` panic that is its body. That is the expected
state of an interface release. They turn green one at a time as bodies land.

Three assertions in `tests/fixedpoint_tests.nv` say that an operation traps,
and `todo()` panics, so those three pass today while meaning nothing. That is
a defect in the test runner rather than a property of this package, and the
assertions are written the way they should be written once the bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `fxq16_16.FxQ16_16`, `fxq1_15.FxQ1_15`, `fxq8_24.FxQ8_24`, `fxuq16_16.FxUQ16_16`, `fixedpoint.FxError` | declared |
| `fxq16_16`: 31 functions over `FxQ16_16` | no |
| `fxq1_15`: 33 functions, including the two conversions to and from Q16.16 | no |
| `fxq8_24`: 33 functions, including the two conversions to and from Q16.16 | no |
| `fxuq16_16`: 31 functions, including the two conversions to and from Q16.16 | no |
| `fxmath.sqrt`, `.invsqrt` | no |
| `fxmath.sin`, `.cos`, `.tan`, `.atan`, `.atan2` | no |
| `fxmath.exp`, `.ln`, `.log2` | no |
| `fxmath.powf`, `.powi` | no |
| `fxmath.floor`, `.ceil`, `.round`, `.trunc`, `.abs` | no |
| `fixedpoint.format_raw`, `.parse_raw`, `.format_name` | no |
| `fixedpoint.format_q16_16` to `.parse_uq16_16`, four pairs | no |
| `fixedpoint.FxError.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
