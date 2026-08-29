# llhttp

HTTP/1.1, parsed for [sysl](https://github.com/sysl-lang/sysl) — bound to
[llhttp](https://github.com/nodejs/llhttp), the parser Node.js is built on.

```
dependencies {
  llhttp { git = "github.com/sysl-lang/llhttp", version = "0.1.0" }
}
```

```sysl
import sh.sysl.llhttp.requests

val r = requests((req) ->
    print(req.method + " " + req.target)
    print(req.header("host").unwrap_or("no host")))

r.feed(bytes)?
```

**Nothing has to be installed.** llhttp is three C files and a header, vendored here, so there is no
`@link`, no `pkg-config` requirement and no library for a machine to be missing. What it needs is a
C library — see *Where this builds* at the end.

## Why a binding rather than a parser written in sysl

Recognising a well-formed request is not the hard part. What is hard is the malformed one, and
specifically the malformed one that is malformed *on purpose*: a request that one hop reads as
having one body and the next reads as having another is a request-smuggling primitive, and it is how
a cache gets poisoned and an authorization check gets walked past.

llhttp is strict about those by construction — it is generated from a state machine rather than
written by hand, and it is what Node has been shipping and hardening for years. Three of them are
tests in this package rather than paragraphs:

| what is sent | what happens |
|---|---|
| `Content-Length: 5` and `Content-Length: 6` | refused, `HPE_INVALID_CONTENT_LENGTH` |
| `Content-Length` beside `Transfer-Encoding: chunked` | refused |
| a chunk size that is not a number | refused, `HPE_INVALID_CHUNK_SIZE` |

`Fault.suspicious()` is the one line of judgement this package offers about them: llhttp has already
refused whichever it answers, and what the answer is for is deciding whether a refusal is worth a log
line. A malformed request from a browser is noise; one of these is somebody trying it on.

## Two tiers, and most programs want the second

```
sh/sysl/llhttp/
    parser.sysl      Parser -- bytes in, events out. Nothing is accumulated and nothing is copied
    request.sysl     Requests -- whole requests, assembled, with limits on how much is held
    error.sysl       Stop and Fault
    c/c.sysl         llhttp as C declares it: the externs, the sizes, the enumerations
    c/llhttp.[ch]    llhttp itself, from the release/v9.4.3 tag, unmodified
package.hocon
```

### `Requests` — whole requests

This is what a server writes against. It concatenates the spans, keeps the header field and value in
step, files each pair, and holds the body — and it puts a **bound** on how much of that it will
hold, which is the part that is not convenience:

```sysl
val r = requests((req) -> serve(req), Limits(8192, 1048576))
```

A server that grows a buffer for as long as bytes keep arriving is one connection away from being
over, and it does not take an attacker — a file upload sent to a JSON endpoint looks exactly the
same. The defaults are 8 KiB of head and 1 MiB of body; exceeding one is a `Refusal`, delivered
where a parse failure would be:

```sysl
r.feed(chunk) match
    Ok(_) -> ()
    Err(BodyTooLarge(n)) -> respond(413)
    Err(HeadTooLarge(n)) -> respond(431)
    Err(Malformed(f)) -> respond(400)
```

A server that streams a large upload rather than holding it uses `Parser` directly, which is the
tier below and holds nothing at all.

### `Parser` — the event stream

One handler and a `match`, rather than the twenty-five separate callbacks llhttp's settings struct
is made of:

```sysl
val p = request_parser((e) -> e match
    Url(bytes) -> ...
    HeaderField(bytes) -> ...
    Body(bytes) -> ...
    MessageComplete -> ...
    _ -> ())

p.execute(chunk)?
```

**A slice handed to a handler is the caller's own buffer**, valid for the call and not after it —
the same contract libuv's read callback has. Nothing is copied, by llhttp or by this.

**A field can arrive in pieces**, and this is the thing to design around rather than to discover.
`HeaderValue` is a span callback, so a value split across two reads arrives as two events with a
`HeaderValueComplete` after the last. That is not an edge case for later: it is what happens the
first time a request straddles two reads off a socket, which under load is most of them.

## A server, whole

Using nothing but `sysl.posix.net` and this package:

```sysl
import sh.sysl.llhttp.{requests, Request}
import sysl.posix.net.{resolve_passive, socket}

private struct Out
    reply: string
    done: bool
end Out

private respond(o: &Out, r: &Request)
    val body = r.method + " " + r.target

    o.reply = "HTTP/1.1 200 OK\r\nContent-Length: " + str(body.bytes.len) +
              "\r\nConnection: close\r\n\r\n" + body
    o.done = true

val addrs = resolve_passive(8080).expect("an address")
var listener = socket(addrs[0]).expect("a socket")

listener.reuse_address().expect("reuse")
listener.bind(addrs[0]).expect("bound")
listener.listen().expect("listening")

loop
    var conn = listener.accept().expect("accepted").0
    val o: &Out = Out("", false)
    val rq = requests((r) -> respond(o, r))

    var scratch: [4096]u8 = [0; 4096]

    while !o.done
        val n = conn.recv(scratch).expect("read")

        if n == 0
            rq.finish().expect("finished")
            break

        rq.feed(scratch[0..<n]) match
            Ok(_) -> ()
            Err(e) ->
                o.reply = "HTTP/1.1 400 Bad Request\r\nContent-Length: 0\r\n\r\n"
                o.done = true

    conn.send_all(o.reply.bytes).expect("wrote")
    conn.close().expect("closed")
```

That is a real program: it was built and driven with `curl` and with a client that dribbled one
request across four TCP segments, splitting the URL, a header name and a header value each in the
middle. All of them answered correctly.

**The callback cannot name the parser it belongs to**, which is worth knowing before writing one: a
closure in sysl captures by value, and the parser does not exist yet when the handler is written. A
box both of them hold is how the knot is untied — `Out` above, and it is the arrangement a real
handler wants anyway, since the thing it acts on is its own state.

## How the binding is arranged

**There is no shim, and llhttp is the second package in this org that needs none.** Every name
llhttp exports is a real symbol — there is no `static inline` in the header, and the constants are
`enum`s, which a `c const` block reads as readily as a macro. The two things that usually force a
line of C are handled by asking the C compiler instead:

- **A callback finds its way home from the parser's own address.** llhttp hands a callback the
  parser and nothing else. There is a `data` field for the caller's word and it is unused here: the
  parser's storage is a *field* of the sysl `Parser`, so subtracting that field's offset from the
  address llhttp gave arrives at the sysl value. No setter, no C, and nothing to keep in step.
- **The chunk size is read at an offset the header computes.** `llhttp_t.content_length` has no
  accessor, so `c const "offsetof(llhttp_t, content_length)"` is what finds it — which follows a
  reordering of the struct rather than being broken by one.

**`llhttp_settings_t` is declared field by field in sysl** rather than left opaque, because every one
of its twenty-five members has to be written and llhttp offers no setter for any of them. A struct
of nothing but pointers has a layout there is nothing to get wrong; what could go wrong is a
*count*, so `sizeof(Settings)` is asserted against the C's `sizeof(llhttp_settings_t)`. An llhttp
that grew a twenty-sixth callback fails that test rather than leaving a pointer uninitialised.

**A parser is a `&Parser`** for the same reason every callback-driven binding in this org uses one:
the address llhttp was given has to still be the sysl value when the callback runs, and a box does
not move.

### The one trap worth writing down

**llhttp has two pause mechanisms and using the wrong one does nothing at all.** `llhttp_pause()` is
for *outside* a callback; a callback has to **return** `HPE_PAUSED`, and the header says so in a note
that is easy to read past. Written the obvious way — a `pause()` that calls `llhttp_pause` — a
handler that pauses parses the whole buffer and reports `Done`, with no error anywhere.

`Parser.pause()` does the right one either way: it raises a flag when it is called from inside a
handler, which the trampoline turns into that return value, and calls `llhttp_pause` when it is not.
A caller never has to know.

## Where this builds

**Any target with a C library**, which is the interesting set: the Pico SDK carries newlib, and so do
Zephyr and FreeRTOS applications. `api.c` includes `<stdio.h>`, `<stdlib.h>` and `<string.h>` — the
first for a debug printer compiled whether or not it is called, the second for the two allocating
calls this binding does not bind, the third for one `memset`.

**Not a bare `thumb-freestanding`**, where clang supplies `stdint.h` and `stddef.h` and none of those
three. This is the same position `qrcodegen` and `qoi` are in, and it is a property of the vendored
C rather than of the binding.

**llhttp itself allocates nothing.** `llhttp_alloc` and `llhttp_free` are the only calls that would
and neither is bound; every byte the parser writes goes into storage this package owns. The `heap`
capability in `package.hocon` is for the `&Parser` box, not for llhttp.

## Testing

```
sysl test .
```

31 tests: the sizes and offsets against what clang measures, the ordinary requests, a request fed one
byte at a time, chunked bodies with trailers, pipelining, upgrade, pause and resume, `skip_body` for
a response to `HEAD`, the three smuggling refusals, and both limits.

## Version

llhttp `release/v9.4.3`, vendored unmodified. `llhttp.c` is generated by llhttp's own build from a
TypeScript description of the state machine, which is why it is ten thousand lines and why it is
never edited here — updating means replacing the three files and running the suite, where the size
assertions are what notice a change in the structs.

MIT, both this and llhttp.
