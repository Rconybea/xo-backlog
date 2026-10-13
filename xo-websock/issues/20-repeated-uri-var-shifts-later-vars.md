# 20 -- a repeated ${var} in a uri pattern binds later variables to the wrong capture

Status: diagnosed
Type: bug
Raised: 2026-10-12, found reading `compile_pattern` while deciding how to print `uri_regex_` (`issues/16`)

For a pattern with a variable repeated before another variable, the later
variable gets the wrong part of the uri.

## Diagnosis

`compile_pattern` (`xo-websock/src/websock/DynamicEndpoint.cpp`) emits one
capture group per `${..}` occurrence, but records each NAME once in
`var_v_` (the `std::find` before `push_back`: "a variable repeated in the
pattern is reported once, at its first position").  `http_response` then
reads captures by `var_v_` position:

```cpp
for (size_t i = 0, n = this->var_v_.size(); i < n; ++i)
    vars.push_back(this->var_v_[i], match[1 + i]);
```

`/x/${a}/${a}/${b}` has groups `a, a, b` but `var_v_ = {a, b}`: `b` reads
group 2 -- the second `a`'s text.  The comment there states the intent ("A
repeated variable takes its first occurrence's value"), which holds for `a`;
it is the variables AFTER the repeat that shift.

Confirmed 2026-10-12 against the real code -- a probe appended to
`xo-websock/utest/UrlRouter.test.cpp` (then removed):

```cpp
TEST_CASE("probe-repeated-var-shifts-later-vars", "[probe]")
{
    auto ep = DynamicEndpoint::make_http("/x/${a}/${a}/${b}",
        [](HttpRequest const & req) {
            return HttpResponse(HttpStatus(200), ContentType::text,
                                std::string(req.vars().lookup("a")) + "," + std::string(req.vars().lookup("b")));
        });
    CHECK(ep->http_response("/x/1/2/3").body() == "1,3");
}
//  FAILED: CHECK( got == "1,3" )  with expansion: "1,2" == "1,3"
```

Only `http_response` reads a URI's captures (`grep -n 'match\[' xo-websock/src/websock/*.cpp`:
the other hits, `DynamicEndpoint.cpp:47-66`, are `compile_pattern` parsing the pattern itself).

## Proposed fix

Record, per variable, the capture group of its first occurrence (e.g. a
`std::vector<std::size_t> var_group_v_` beside `var_v_`, filled in
`compile_pattern`), and read `match[var_group_v_[i]]`.  The probe above
becomes the regression test.

Worth deciding alongside (unverified, a policy question): should a repeated
variable REQUIRE equal values -- `/x/${a}/${a}` matching `/x/1/1` but not
`/x/1/2` (a regex backreference) -- or keep "first occurrence wins"?

**Done when:** the probe passes as a test in `UrlRouter.test.cpp`, and the
existing url-router tests stay green.
