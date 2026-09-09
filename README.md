# kotoba-lang/org-ieee-wc — POSIX `wc`, as a Kotoba command binary

`wc` from IEEE Std 1003.1 for one operand, written in `.kotoba` and compiled
to a standalone native executable.

```sh
./wc FILE        # lines, words, bytes
./wc -l FILE     # lines
./wc -w FILE     # words
./wc -c FILE     # bytes
```

## The format is `" %7d"`, and that took a fixture to prove

macOS `/usr/bin/wc` prints each count with an **unconditional leading space**
and the number right-aligned in **seven** columns. A fixed eight-wide field
looks identical until a count reaches eight digits — below that both produce
eight characters.

Measured 2026-09-10: padding to a fixed 8 passed **fourteen of fifteen**
cases. Only a 10,000,000-byte operand separates them:

```
correct      " 10000000 …"     (space + 8 digits)
fixed %8d    "10000000 …"      (no space)
```

That fixture is in the suite for exactly this reason, and `-c` is the case
that uses it: bytes need no walk, so the boundary costs one read.

## Measured against the system utility

`test/wc_test.cljs` compiles the guest, packages it, **runs the binary**, and
compares bytes against `/usr/bin/wc`. Sixteen cases, all byte-identical.

Each earns its place:

| fixture | what it separates |
|---|---|
| two lines | the three counts together |
| no trailing newline | lines are **newlines**: answers 0, not 1 |
| empty | three zeroes, not a blank line |
| UTF-8 | bytes are bytes (13) and words are words (2) |
| blank lines | empty lines count as lines and not as words |
| tabs | a tab separates words the way a space does |
| 10,000,000 bytes | the eight-digit format boundary |

Verified to fail as well as pass: counting word *characters* instead of word
*starts* fails three cases, dropping the tab from the whitespace set fails
two, and the fixed-width format fails exactly the boundary case and nothing
else.

## Cost is packaged, not supplied

Counting words walks the operand one **code point** at a time, so the guest
recursion is as long as the file. Fuel and the string arena are constants of
the packaged binary (`--fuel`, `--string-pool`) — like the grant and the
filesystem scope, a caller cannot raise them. The default 512 fuel counts
almost nothing; package with a budget that matches the files you mean to
count.

`-c` computes only bytes, `-l` only lines, `-w` only words. That is not a
micro-optimisation: computing words for `-c` would make the cheapest question
the most expensive one, and would put the eight-digit fixture out of reach of
a test that has to fit in fuel.

## Capabilities

`:cli/args` (38), `:fs/app-data` (35), `:io/write` (37).

## What this is not

One operand. `/usr/bin/wc` with several prints a `total` line, which this
does not. No `-m` (characters), and no reading standard input — there is no
stdin capability.
