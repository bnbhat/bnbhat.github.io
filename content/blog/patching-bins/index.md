---
title: "The Byte That Wasn't There: Patching a Binary You Don't Have the Source For"
summary: A walkthrough of diagnosing a vague runtime error back to a stale lookup table inside a prebuilt .so, and patching it byte-for-byte without the source.
date: 2026-10-10
cardimage: patching-bins.png
featureimage: patching-bins.png
caption: ""
authors:
  - bnbhat: author.jpeg
---

## The mystery

It started with a single, unhelpful line of output:

```
ERROR 0xe: unable to get domain struct 6
```

A test tool was failing against a brand-new hardware subsystem — one that,
by every other signal on the system, was working perfectly fine. A daemon
responsible for that exact subsystem was running, and its logs showed it
successfully attaching and doing its job. Yet the moment the test harness
tried to use that same subsystem, it fell over with a generic "bad
parameter" error and gave up.

This is a familiar shape of bug for anyone who's worked on embedded systems,
SDKs, or any codebase with a "test tools" folder nobody has touched in years:
**the error message doesn't match the system's actual behavior.** Something,
somewhere, disagrees with reality, and the only way out is to go find out
exactly where.

What follows isn't really about this one bug. It's a walkthrough of a
technique that applies anywhere you hit a wall like this: *a prebuilt
binary, no source in sight, and a failure that only it believes in.*

## Step 1: Don't trust the first theory

The tempting first explanation was the simplest one — "this hardware doesn't
support that subsystem." It's a satisfying answer, because it requires no
further work; you just stop using it and move on.

But satisfying isn't the same as correct. The system journal told a
different story: a daemon for that exact subsystem had started up moments
earlier and was working without complaint. If the hardware genuinely didn't
support it, that daemon wouldn't exist, let alone run cleanly.

**Lesson:** when a failure contradicts other evidence on the same system,
trust the evidence, not the error message. The daemon logs were first-hand
proof the capability existed; the test tool's complaint was just a
complaint.

## Step 2: Watch syscalls before you read code

Rather than open an editor, the next move was to watch what the failing
process actually *did* at the kernel boundary, using `strace`. This is
cheap, non-destructive, and tells you the truth about execution order
instead of the intended order:

```
openat(AT_FDCWD, "/dev/dma_heap/system", O_RDONLY|O_CLOEXEC) = 5
...                                   (unrelated config parsing)
write(1, "Creating sequence of numbers fro"..., 43)
write(1, "\nERROR 0xe: unable to get domain"..., 42)
```

Critically, **there was no `open` or `ioctl` on any device file in
between.** No attempt to talk to the hardware at all. That one observation
collapsed the search space enormously: whatever was failing, it was failing
*before* anything reached the driver or the hardware. It had to be pure
userspace logic — a check, a lookup, a table — deciding to give up early.

## Step 3: Grep the source before you suspect the source

With "it's a userspace check" as the working theory, the natural next step
was to search the library's own source for the exact error string:

```
$ grep -r "unable to get domain struct" src/ inc/
(no matches)
```

That's a useful negative result, not a dead end. If the string isn't
anywhere in the source you have, whatever's printing it isn't *your* code —
it's something else linked into the same process. Here, the test harness
turned out to be a thin loader: it `dlopen()`s a handful of prebuilt `.so`
payload files and calls a known entry point in each. The actual logic — and
the bug — lived entirely inside binaries nobody had the source for anymore.

This happens more often than people expect: vendor test suites, legacy
plugins, old demo binaries shipped with an SDK, third-party libraries
bundled into a build. The investigation doesn't stop here; it just changes
tools.

## Step 4: Binaries still talk, if you ask nicely

When source runs out, standard toolchain utilities can read a binary for
you, for free:

```
$ strings -a payload.so | grep -i "domain struct"
ERROR 0x%x: unable to get domain struct %d
```

Found it — inside the prebuilt payload, confirming it wasn't the real
runtime library talking.

```
$ file payload.so
payload.so: ELF 64-bit LSB shared object, ARM aarch64, ... with debug_info, not stripped
```

"Not stripped" is a huge win — function and variable *names* survived, not
just addresses:

```
$ nm -C payload.so | grep -i domain
0000000000002dc4 T get_domain
0000000000002e44 T is_valid_domain_id
```

(`T` = a function in `.text`; uppercase means globally visible. `nm` uses a
handful of these letters — `D`/`d` for data, `B`/`b` for bss, `U` for
undefined/external, `W`/`w` for weak symbols — enough to turn a wall of
addresses into a readable map.)

Disassembling the function named `get_domain` (`objdump -d -C`, using the
architecture-appropriate cross tool when the host one can't decode the
target ISA) showed a simple linear search over a **5-entry table**, 16 bytes
per entry, returning `NULL` if nothing matched. Tracing the caller a little
further showed exactly what happens on a miss:

```asm
bl   get_domain          ; x0 = get_domain(domain_id)
cmp  x0, #0x0
b.ne continue            ; found -> proceed normally
mov  w0, #0xe             ; nErr = 0xe  (a generic "bad parameter" code)
bl   printf               ; "unable to get domain struct %d"
b    bail                 ; give up, no device ever touched
```

That matched the error text, the error code, *and* the "no syscalls"
observation from step 2 — exactly. Root cause located with certainty, not
guesswork.

## Step 5: Find the table and dump it

The table itself was loaded indirectly through a relocation. `readelf -r`
named the actual symbol; `readelf -x` then dumped its raw bytes:

```
0x6110: id=0  "&_dom=adsp"
0x6120: id=1  "&_dom=mdsp"
0x6130: id=2  "&_dom=sdsp"
0x6140: id=3  "&_dom=cdsp"
0x6150: id=4  "&_dom=cdsp1"
```

Five known IDs, baked into the binary at build time. The ID the test was
being asked to use simply wasn't on the list — not a hardware limitation, not
a driver bug, just a stale lookup table from before that subsystem existed.

## Step 6: Patch it like a surgeon

Once you know exactly which bytes are wrong and exactly why, you can choose
to fix it directly in the compiled artifact. Binary patching has a bad
reputation, usually from people doing it carelessly. Done carefully, it's
precise and reversible, if you follow a few rules:

1. **Never edit in place.** Work on a copy; keep the original backed up
   somewhere the running system won't pick it up by accident.
2. **Change bytes, not structure.** Overwriting existing bytes with new
   values of the *same length* is safe. Growing or shrinking a data
   structure shifts everything after it in the file and cascades into a full
   relink you can't do by hand without the original build system.
3. **Reuse, don't append.** An unused table slot — a legacy option, a value
   the current hardware doesn't have — is much safer to repurpose than
   trying to insert a new one. Just confirm first that nothing else in the
   binary depends on that slot's old meaning (a quick `strings`/disassembly
   search is cheap insurance).
4. **Verify mechanically.** Dump the bytes before and after with the same
   tool and diff them. You want to see *exactly* the bytes you intended to
   change, and nothing else.
5. **Know what you're trading away**, and write it down. Here, sacrificing
   one legacy table entry meant that option could no longer be tested with
   this particular binary — an acceptable trade on hardware that never had
   it anyway, but a conscious one.

The actual patch was a five-line script: open a copy, seek to the known
offset, overwrite the 16-byte record with the new ID and name, and confirm
with the same `readelf -x` dump used to find the table in the first place.
One byte changed the ID, twelve more replaced the name — the rest of the
file, untouched.

## Step 7: Write it down

The most important, most skipped step: once you've found the problem and
fixed it, document the investigation, not just the fix. A patched binary
with no explanation is a landmine for the next person — quite possibly
future-you — who updates the vendor package, wonders why a previously-fixed
test is suddenly broken again, and has no idea a byte-level patch ever
existed.

A good writeup includes the symptom and how to reproduce it, the evidence
that ruled out the wrong explanations, the tools and commands used in order
(so the investigation is repeatable, not just the conclusion), the exact
bytes changed at exact offsets in exact files, and the caveats: what you
can't test anymore, and why this is a workaround rather than a real fix.

## The takeaway

Strip away the specific error message and the specific subsystem, and what's
left is a pattern that shows up constantly in systems work:

> Something fails with a vague error. The system around it clearly supports
> what's being asked. The source for the thing that's actually complaining
> is nowhere to be found. The fix is in there somewhere, encoded in bytes
> instead of text.

The process that gets you from "vague error" to "exact byte, exact offset,
exact reason" is always the same shape:

1. Trust the system's observable behavior over the error text.
2. Watch syscalls before you read code — it tells you *where* to look.
3. Grep the source you have, and take a negative result seriously.
4. When source runs out, let `strings`, `nm`, `objdump`, and `readelf` read
   the binary for you.
5. Patch narrowly, on a copy, with a mechanical before/after verification.
6. Document the investigation as carefully as the fix.

The next cryptic error you hit probably won't involve a DSP subsystem. It
might be a plugin, a vendored library, a decade-old shared object, a config
parser that only half-understands a newer file format. The tools and the
discipline are the same either way — the binary already knows the answer,
you just have to ask it the right way.
