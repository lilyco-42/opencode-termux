# opencode-termux

Fix for running [opencode](https://opencode.ai) on Termux with a glibc setup
(e.g. the `termux-glibc` repository that provides `glibc`, `glibc-runner`,
`grun`, etc.).

## Symptom

Every command run through opencode's shell tool fails with:

```
CANNOT LINK EXECUTABLE "/data/data/com.termux/files/usr/bin/zsh":
"/data/data/com.termux/files/usr/glibc/lib/libc.so" has bad ELF magic: 2f2a2047
```

The magic `2f2a2047` is `/* G` — the start of a GNU ld **linker script**, not
an ELF object.

## Cause

The opencode binary is linked against glibc, so its launcher has to start it
through the glibc loader and expose the glibc libraries. The common (broken)
launcher does this by exporting `LD_LIBRARY_PATH`:

```sh
#!/data/data/com.termux/files/usr/bin/sh
unset LD_PRELOAD
export LD_LIBRARY_PATH=/data/data/com.termux/files/usr/glibc/lib:$LD_LIBRARY_PATH
exec /data/data/com.termux/files/usr/glibc/lib/ld-linux-aarch64.so.1 \
    "$HOME/.opencode/bin/opencode" "$@"
```

`LD_LIBRARY_PATH` is **inherited by every child process**. opencode spawns the
user's shell (from `$SHELL`, typically the normal bionic `/usr/bin/zsh`). When
Android's bionic linker starts that binary it searches `LD_LIBRARY_PATH` first
and finds `.../glibc/lib/libc.so` — a linker script — and aborts with the
"bad ELF magic" error above.

## Fix

Pass the library search path to the loader with `--library-path` instead of
exporting it. The loader honors it for opencode and its dependencies, but it is
never written into the environment, so child processes stay clean:

```sh
#!/data/data/com.termux/files/usr/bin/sh
unset LD_PRELOAD
unset LD_LIBRARY_PATH
exec /data/data/com.termux/files/usr/glibc/lib/ld-linux-aarch64.so.1 \
    --library-path /data/data/com.termux/files/usr/glibc/lib \
    "$HOME/.opencode/bin/opencode" "$@"
```

## Install

Overwrite the launcher (usually `/data/data/com.termux/files/usr/bin/opencode`)
with [`opencode`](./opencode):

```sh
cp opencode "$PREFIX/bin/opencode"
chmod +x "$PREFIX/bin/opencode"
```

Then restart opencode. `echo test` in the shell tool should now work.

> Note: a package update of opencode may overwrite this launcher; re-apply the
> fix afterwards.
