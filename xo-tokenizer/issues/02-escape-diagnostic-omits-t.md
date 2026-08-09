# 02 — string-escape diagnostic under-reports the accepted set (omits `t`)

Status: diagnosed
Type: bug

The tokenizer accepts five escapes inside a string literal. Its error message
for a rejected one names four:

```cpp
/* xo-tokenizer/include/xo/tokenizer/tokenizer.hpp:514-541 */
case '\\': ...   case 'n': ...   case 't': ...   case 'r': ...   case '"': ...
default:
    return result_type::make_error_consume_current_line
        (__FUNCTION__,
         "expecting one of n|r|\"|\\ following escape \\",   /* :538 -- no t */
         ...);
```

So `"a\tb"` reads fine, but anyone who mistypes an escape is told `\t` is not
allowed.

Measured 2026-08-08:

```bash
# five cases in the switch
sed -n '514,541p' xo-tokenizer/include/xo/tokenizer/tokenizer.hpp | grep -E '^\s+case'
#   case '\\':  case 'n':  case 't':  case 'r':  case '"':

# and t genuinely works -- pinned by an existing test
grep -n 'tab to the right' xo-tokenizer/utest/tokenizer.test.cpp
#   :191  feeds "tab to the right [\t]..." and expects a real tab in the token
```

## Why it matters more than a typo

The message is the only place the accepted set is written down *as a set*, so
it reads like the specification — and it was read that way.
`.xo-backlog/xo-object/issues/01` asserted "xo-reader accepts `\n \r \" \\`"
on the strength of this string, and built an argument on the escape set being
narrower than ppsink's. Corrected there on 2026-08-08.

That misreading had a real consequence: it made `\t` look like it could not
round-trip, when in fact it can, which understated the value of
`.xo-backlog/xo-ppsink/issues/04`.

**Files:**
- `xo-tokenizer/include/xo/tokenizer/tokenizer.hpp:538` — the message
- `xo-tokenizer/include/xo/tokenizer/tokenizer.hpp:514-541` — the switch it
  should agree with
- `xo-tokenizer/utest/tokenizer.test.cpp` — where escape handling is pinned

**Done when:**
- the diagnostic names every escape the switch accepts
- a test asserts the rejection message for an unsupported escape, so the two
  cannot drift apart again silently

## Notes

Worth fixing as *derived rather than hand-maintained* if that is cheap here —
the same principle the milestone machinery uses (`Milestone:` lines queried
rather than a hand-kept list). A hand-written string beside a switch is exactly
the shape that rots: nothing fails when someone adds a case.

If deriving is awkward, the test above is the cheap substitute — it fails when
the switch gains a case and the message does not.
