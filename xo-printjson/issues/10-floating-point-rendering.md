# 10 — floating-point json: lossy precision, and undeliberate choices

Status: diagnosed
Type: bug
Raised: 2026-10-16 (RC: "choices made there weren't particularly deliberate"),
found while landing step 1 of `.xo-backlog/xo-printjson/issues/01`

`JsonPrinter_floatingpoint<T>` (float and double) writes a finite value with
a bare `*p_os << *x`, so a json number gets whatever the ostream's format
state is -- by default, 6 significant digits.  A double does not survive a
round trip.

## What is there

```bash
cd "$(xo-sdlc --umbrella2)"
grep -n 'class JsonPrinter_floatingpoint' -A35 xo-printjson/src/printjson/PrintJson.cpp
#   finite:      *p_os << *x;
#   non-finite:  NaN / Infinity / -Infinity  ("javascript-friendly")
grep -n 'provide_floatingpoint_printer<' xo-printjson/src/printjson/PrintJson.cpp
#   float, double
```

Unchanged since the 2026-06-06 subrepo import -- the earliest history this
tree has, so no recorded reason for any of it:

```bash
git log --format='%h %ad %s' --date=short -S'Infinity' -- xo-printjson/src/printjson/PrintJson.cpp
```

xo-kalmanfilter's matrix/vector printers route each element through the
same printer (`state.print(Reflect::make_tp(&(*pv)[i]))`,
`xo-kalmanfilter/src/kalmanfilter/EigenUtil.cpp`), so one fix covers them.

## Observed

A default-constructed ostream -- which is what the printer writes into, both
before `issues/01` step 1 (every caller in the tree passed an unconfigured
stream) and after it (`JsonPrintState`'s own `os_`).  Measured with a
standalone `std::ostringstream << double` (2026-10-16, gcc 14.3):

| value | written | |
|---|---|---|
| `1.5` | `1.5` | |
| `1.23456789` | `1.23457` | **lossy** |
| `123456789.0` | `1.23457e+08` | **lossy** -- an integer-valued double |
| `9007199254740993.0` | `9.0072e+15` | **lossy** |
| `2.0/3.0` | `0.666667` | **lossy** |
| `3.0` | `3` | valid; indistinguishable from an int |
| `-0.0` | `-0` | valid json |
| `1e21` / `1e-7` | `1e+21` / `1e-07` | valid json |
| `0.1f` (float) | `0.1` | |

## The choices, none yet made deliberately

1. **Precision.**  Shortest representation that round-trips
   (`std::to_chars(first, last, x)` with no precision argument -- C++17,
   locale-free, exact) is the obvious candidate; `max_digits10` with `%g`
   also round-trips but writes noise (`0.1` -> `0.10000000000000001`).
   float should round-trip as float, not be widened to double first.
2. **Non-finite values.**  Today `NaN` / `Infinity` / `-Infinity` -- NOT
   json: browsers' `JSON.parse` rejects them, and the introspect page parses
   every frame with it (`xo-websock/example/introspect/mount-origin/introspect.js`,
   `JSON.parse(ev.data)`).  Python's `json.loads` accepts them by default.
   Options: keep the extension; `null` (loses which); a string (`"NaN"`,
   ambiguous with a real string); or make it a PrintJson setting.
3. **Integral-valued doubles.**  `3` vs `3.0`: both valid; `3.0` keeps
   "this was floating point" visible to a reader, `3` is what most
   serialisers write.
4. **Locale.**  `operator<<` honours the stream's locale (a `,` decimal
   separator would corrupt the document).  `std::to_chars` does not consult
   a locale, which settles this if (1) chooses it.

## Evidence and recommendations (2026-10-17, prepared while RC away -- decisions are RC's)

Measured: `std::to_chars(first, last, x)` (shortest round-trip) vs today's
`%.6g` (gcc 14.3), and two real consumers (node 22.19 -- the same engine
family as the introspect page's browser -- and python 3.12).

| value | shortest | today | round-trips (shortest) |
|---|---|---|---|
| `1.5` | `1.5` | `1.5` | yes |
| `1.23456789` | `1.23456789` | `1.23457` | yes |
| `123456789.0` | `123456789` | `1.23457e+08` | yes |
| `0.1` | `0.1` | `0.1` | yes |
| `2.0/3.0` | `0.6666666666666666` | `0.666667` | yes |
| `9007199254740992.0` | `9007199254740992` | `9.0072e+15` | yes |
| `5e-324` | `5e-324` | `4.94066e-324` | yes |
| `0.1f` (float) | `0.1` | `0.1` | yes -- but widened to double first: `0.10000000149011612` |

Consumers:

```
node:   JSON.parse("NaN")      -> throws: "NaN" is not valid JSON   (same for "Infinity")
        JSON.stringify(NaN)    -> null ;  JSON.stringify([Infinity]) -> [null]
        JSON.parse("3.0")      -> 3      (indistinguishable from "3")
python: json.loads("NaN")      -> nan    (accepts by default)
        json.dumps(nan, allow_nan=False) -> ValueError: not JSON compliant
```

Recommendations, one per choice:

1. **Precision: shortest round-trip, `std::to_chars(first, last, x)` with
   no format or precision.**  Every value above round-trips; values that
   were already short (`0.1`, `1.5`) print exactly as today, so only the
   lossy ones change.  **`float` formatted as `float`**, not widened: the
   `to_chars(float)` overload gives `0.1`.
2. **Non-finite: `null`.**  It is what JavaScript's own `JSON.stringify`
   writes for NaN and +-Infinity, and it parses everywhere.  The cost: a
   consumer cannot tell NaN from Infinity from a real null.  Today's
   `NaN` / `Infinity` make the introspect page's `JSON.parse` throw on any
   frame containing one -- the whole frame is lost, not just the value.
   If a consumer needs the distinction (python, which accepts the
   extension), a `PrintJson` setting could select the extension -- but only
   if someone asks.
3. **Integral-valued doubles: `3`**, what `to_chars` and `JSON.stringify`
   both write.  `JSON.parse` cannot tell `3` from `3.0` anyway, and the
   declared type travels separately (a member's `_canonical_type_`).
4. **Locale: settled by (1)** -- `to_chars` does not consult a locale.

Where it lands: `JsonPpSink::number(double)` and `number(float)`
(`.xo-backlog/xo-printjson/issues/11`) -- one place, plus
`JsonPrinter_floatingpoint`'s non-finite branch, which today calls `raw()`.
Changes pinned output: xo-kalmanfilter's `json_eigen.test.cpp` pins `NaN` /
`Infinity`, and any golden or pin holding a lossy double.

## Interaction with issues/01

Step 1 of `issues/01` made the printer write into `JsonPrintState`'s own
ostream instead of the caller's -- so format flags a caller set (precision,
`std::fixed`) no longer reach it.  No caller in the tree set any, so output
did not change; but it means the fix belongs IN the printer, not in caller
stream configuration.  Once the printer writes through `state.sink()`
(`issues/01`'s migration), `to_chars` into a small stack buffer and one
`put()` is the natural shape.

**Done when:**
- (1)-(4) decided, and recorded here with the reason
- a double round-trips: a test printing values from the table above, parsed
  back (`std::from_chars`), compares equal -- including `9007199254740993.0`
  and `2.0/3.0`
- float prints as float (`0.1f` -> `0.1`, not `0.10000000149011612`)
- the non-finite cases pinned to whatever (2) decides
