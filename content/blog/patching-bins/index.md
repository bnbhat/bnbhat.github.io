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

A test tool failed with this error:

```
ERROR 0xe: unable to get domain struct 6
```

The subsystem it was testing looked fine otherwise. Its daemon was running and its logs showed it attaching and working correctly. Only the test harness complained, with a generic "bad parameter" error.

This post isn't really about that one bug — it's about the process for tracking down a failure when you only have a prebuilt binary and no source for it.

## Step 1: Don't trust the first theory

The easy explanation was "this hardware doesn't support that subsystem." But the daemon logs contradicted that — it had started up and was working without issue moments earlier. If the hardware didn't support it, that daemon wouldn't be running cleanly.

When a failure contradicts other evidence on the same system, trust the evidence over the error message.

## Step 2: Watch syscalls before you read code

Instead of opening an editor, I traced the failing process with `strace` to see what it actually did:

```
openat(AT_FDCWD, "/dev/dma_heap/system", O_RDONLY|O_CLOEXEC) = 5
...                                   (unrelated config parsing)
write(1, "Creating sequence of numbers fro"..., 43)
write(1, "\nERROR 0xe: unable to get domain"..., 42)
```

There's no `open` or `ioctl` on any device file between those lines. The failure happens before anything reaches the driver or hardware — so it has to be pure userspace logic: a check, a lookup, a table, deciding to give up early.

## Step 3: Grep the source before you suspect the source

Working theory: a userspace check. Next step, search the library source for the exact error string:

```
$ grep -r "unable to get domain struct" src/ inc/
(no matches)
```

A useful negative result. If the string isn't in the source you have, something else linked into the process is printing it. Here, the test harness turned out to be a thin loader: it `dlopen()`s a few prebuilt `.so` payload files and calls a known entry point in each. The actual logic — and the bug — lives inside binaries nobody has the source for anymore. This is common with vendor test suites, legacy plugins, and old SDK demo binaries.

## Step 4: Binaries still talk, if you ask nicely

With no source, standard toolchain utilities can still read the binary:

```
$ strings -a payload.so | grep -i "domain struct"
ERROR 0x%x: unable to get domain struct %d
```

Confirmed — the string lives inside the payload, not the real runtime library.

```
$ file payload.so
payload.so: ELF 64-bit LSB shared object, ARM aarch64, ... with debug_info, not stripped
```

"Not stripped" means function and variable names survived:

```
$ nm -C payload.so | grep -i domain
0000000000002dc4 T get_domain
0000000000002e44 T is_valid_domain_id
```

(`T` = function in `.text`, uppercase = globally visible. Other common letters: `D`/`d` data, `B`/`b` bss, `U` undefined/external, `W`/`w` weak.)

Disassembling `get_domain` (`objdump -d -C`, using the architecture-appropriate cross tool when the host one can't decode the target ISA) showed a linear search over a 5-entry table, 16 bytes per entry, returning `NULL` on no match. The caller handles a miss like this:

```asm
bl   get_domain          ; x0 = get_domain(domain_id)
cmp  x0, #0x0
b.ne continue            ; found -> proceed normally
mov  w0, #0xe             ; nErr = 0xe  (a generic "bad parameter" code)
bl   printf               ; "unable to get domain struct %d"
b    bail                 ; give up, no device ever touched
```

This matches the error text, the error code, and the "no syscalls" observation from step 2. Root cause confirmed.

## Step 5: Find the table and dump it

The table is loaded indirectly through a relocation. `readelf -r` named the symbol; `readelf -x` dumped its bytes:

```
0x6110: id=0  "&_dom=adsp"
0x6120: id=1  "&_dom=mdsp"
0x6130: id=2  "&_dom=sdsp"
0x6140: id=3  "&_dom=cdsp"
0x6150: id=4  "&_dom=cdsp1"
```

Five known IDs baked into the binary at build time. The ID the test used wasn't on the list — not a hardware limitation, just a stale lookup table from before this subsystem existed.

## Step 6: Patch it like a surgeon

Once you know exactly which bytes are wrong and why, you can patch the compiled artifact directly. Rules that keep it safe and reversible:

1. **Never edit in place.** Work on a copy; keep the original backed up.
2. **Change bytes, not structure.** Overwriting existing bytes with same-length values is safe. Growing or shrinking a structure shifts everything after it and effectively requires a full relink.
3. **Reuse, don't append.** An unused table slot (a legacy option, a value the current hardware doesn't have) is safer to repurpose than inserting a new one. Confirm first that nothing else depends on that slot's old meaning — a quick `strings`/disassembly search is cheap insurance.
4. **Verify mechanically.** Dump the bytes before and after with the same tool and diff them. You should see exactly the bytes you intended to change, nothing else.
5. **Know what you're trading away, and write it down.** Here, sacrificing one legacy table entry meant that option could no longer be tested with this binary — acceptable since the hardware never had it, but worth recording.

The patch itself: open a copy, seek to the known offset, overwrite the 16-byte record with the new ID and name, and confirm with the same `readelf -x` dump used to find the table. One byte changed the ID, twelve more replaced the name — the rest of the file untouched.

## Step 7: Write it down

Document the investigation, not just the fix. A patched binary with no explanation is a landmine for whoever updates the vendor package next and wonders why a previously-fixed test is suddenly broken again.

A good writeup covers:
- The symptom and how to reproduce it
- The evidence that ruled out the wrong explanations
- The tools and commands used, in order
- The exact bytes changed, at exact offsets, in exact files
- The caveats: what you can't test anymore, and why this is a workaround rather than a real fix

## The takeaway

The pattern: a vague error, a system that clearly supports what's being asked, and no source for the thing that's actually complaining. The fix is encoded in bytes instead of text.

1. Trust the system's observable behavior over the error text.
2. Watch syscalls before you read code — it tells you where to look.
3. Grep the source you have, and take a negative result seriously.
4. When source runs out, let `strings`, `nm`, `objdump`, and `readelf` read the binary for you.
5. Patch narrowly, on a copy, with a mechanical before/after verification.
6. Document the investigation as carefully as the fix.

The next cryptic error probably won't involve a DSP subsystem — it might be a plugin, a vendored library, or a config parser that only half-understands a newer file format. The tools and discipline are the same either way.
