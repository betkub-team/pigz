# CLAUDE.md — pigz (Windows rebuild fork)

This is a fork of [pigz](https://github.com/madler/pigz) maintained at
`betkub-team/pigz` to produce **static Windows `pigz.exe` / `unpigz.exe` (v2.9)** for use by the
WSL VHDX backup scripts in `d:\WSL\script` (`wsl-backup.ps1` / `restore.ps1`).

## What this fork adds over upstream v2.8
- **Linked against zlib-ng** (zlib-compat, static) → runtime-dispatched AVX2 deflate, faster on
  modern Win11 CPUs while staying portable (no `-march=native`).
- **`--progress` / `--no-progress`** — live meter on **stderr** for both compress and decompress
  (stdout stays a pure data stream for `-c` pipelines). Throttled ~250 ms; shows %, GB/GB, MB/s,
  or bytes+rate when the input size is unknown (stdin/pipe).
- **Windows program-name detection (Patch A)** — `main()` scans for both `/` and `\` and strips a
  trailing `.exe` (case-insensitive), so a binary named `unpigz.exe` auto-decompresses.
- **winpthreads lock-hang avoided** by building with the mingw **14-posix** toolchain (fixed
  winpthreads CV race). No pigz core change needed.

## Build
Use the **`build-pigz`** skill (`.claude/skills/build-pigz/SKILL.md`) — it encodes the full recipe
(toolchain, zlib-ng cmake, the `make` line, the `cp pigz.exe unpigz.exe` workaround, static check,
verify, stage/deploy). Quick version:
```sh
ZNG=/home/admin/projects/zlib-ng/install-win
make CC=x86_64-w64-mingw32-gcc \
  CFLAGS="-O3 -Wall -Wextra -Wno-unknown-pragmas -Wcast-qual -I$ZNG/include" \
  LDFLAGS="-static -static-libgcc -L$ZNG/lib" \
  LIBS="-lm -lpthread $ZNG/lib/libz.a"
cp pigz.exe unpigz.exe   # mingw emits pigz.exe; the Makefile's `ln pigz unpigz` step is expected to fail
```
We are inside Debian13 (WSL). `.exe` cannot run here (no wine) — **runtime verification is
Windows-side**. Deploy target `d:\WSL\script` is reachable at `/mnt/d/WSL/script`.

## Conventions
- **Never `-march=native`** / `WITH_NATIVE_INSTRUCTIONS` — all SIMD lives in zlib-ng's runtime
  dispatch; native flags cause illegal-instruction crashes on older CPUs.
- Static link is mandatory: `objdump -p pigz.exe` must show only `KERNEL32.dll` + `msvcrt.dll`.
- Version string: `#define VERSION "pigz 2.9"` in `pigz.c` (keep header comment + changelog in sync).
- **Don't deploy without Windows-side verification + user confirmation.** Stage version-named copies
  (`pigz-2.9.exe`) first; back up the live binary as `*.v2.8-zlib.bak` before promoting. Leave the
  existing `*.x86-32bit.bak` files untouched.

## Reference docs
- `docs/build-plan.md` — full plan, research findings, rationale (zlib-ng dispatch, the lock-hang
  analysis, all phases).
- `docs/HANDOFF.md` — running status of the rebuild effort.
