---
name: build-pigz
description: Cross-compile a static Windows pigz.exe/unpigz.exe (v2.9) from this repo on Debian mingw-w64, linked against zlib-ng (runtime AVX2). Use when rebuilding the Windows pigz binaries, after editing pigz.c, or when the user mentions building/cross-compiling pigz, the --progress meter, or the winpthreads lock-hang fix.
---

# build-pigz — cross-compile static Windows pigz.exe with zlib-ng

Rebuilds `pigz.exe` / `unpigz.exe` for Windows from `pigz.c` in this repo. Output is a **static**
binary (no DLL deps beyond KERNEL32 + msvcrt), linked against **zlib-ng** for runtime-dispatched
AVX2 deflate, with a `--progress` meter and the winpthreads lock-hang avoided via mingw 14-posix.

All commands run inside Debian13 (this WSL instance is already it). Deploy target is the Windows
folder `d:\WSL\script` (reachable here at `/mnt/d/WSL/script`).

## Layout / prerequisites (one-time, Phase 0–2)
- Toolchain: `gcc-mingw-w64-x86-64` set to the **posix** variant (fixed winpthreads) + `cmake`:
  ```sh
  sudo apt install -y gcc-mingw-w64-x86-64 cmake
  sudo update-alternatives --set x86_64-w64-mingw32-gcc /usr/bin/x86_64-w64-mingw32-gcc-posix
  x86_64-w64-mingw32-gcc --version   # expect "14-posix"
  ```
- zlib-ng static `libz.a` at `/home/admin/projects/zlib-ng/install-win/`. If missing, rebuild:
  ```sh
  cd /home/admin/projects/zlib-ng
  cmake -S . -B build-win \
    -DCMAKE_SYSTEM_NAME=Windows \
    -DCMAKE_C_COMPILER=x86_64-w64-mingw32-gcc \
    -DCMAKE_RC_COMPILER=x86_64-w64-mingw32-windres \
    -DCMAKE_BUILD_TYPE=Release \
    -DZLIB_COMPAT=ON -DBUILD_SHARED_LIBS=OFF \
    -DWITH_RUNTIME_CPU_DETECTION=ON -DWITH_GZFILEOP=ON \
    -DWITH_NATIVE_INSTRUCTIONS=OFF -DZLIB_ENABLE_TESTS=OFF \
    -DCMAKE_INSTALL_PREFIX=/home/admin/projects/zlib-ng/install-win   # absolute, NOT $PWD
  cmake --build build-win -j && cmake --install build-win
  ```
  NEVER `-march=native` / `WITH_NATIVE_INSTRUCTIONS=ON` — illegal-instruction crash on old CPUs.

## Source patches already in pigz.c (Phase 3 — re-derive only if starting from upstream)
1. struct `g`: `length_t in_size; length_t in_seen; int progress;`
2. `show_progress(int final)` helper (after `grumble()`) — throttled ~250 ms `\r` line to **stderr**.
3. `readn()`: after the read loop, `if (g.progress && desc == g.ind) { g.in_seen += got; show_progress(0); }`.
4. Patch A — `main()` program-name scan for both `/` and `\`, strips trailing `.exe` (case-insensitive)
   under `#ifdef _WIN32` so `unpigz.exe` auto-decompresses.
5. `--progress` / `--no-progress` long options in `option()`; `--progress` line in `help()`.
6. `process()`: stdin branch sets `g.in_size = 0; g.in_seen = 0;`. File branch (after `g.ind = open`)
   sets `g.in_size = S_ISREG(st.st_mode) ? (length_t)st.st_size : 0; g.in_seen = 0;`.
7. End of `process()`: after the `verbosity > 1` newline block,
   `if (g.progress) { show_progress(1); putc('\n', stderr); fflush(stderr); }`.

Version string lives at `#define VERSION "pigz 2.9"` (plus header comment + changelog block).

## Build (Phase 4)
```sh
cd /home/admin/projects/pigz
ZNG=/home/admin/projects/zlib-ng/install-win
make clean
make CC=x86_64-w64-mingw32-gcc \
  CFLAGS="-O3 -Wall -Wextra -Wno-unknown-pragmas -Wcast-qual -I$ZNG/include" \
  LDFLAGS="-static -static-libgcc -L$ZNG/lib" \
  LIBS="-lm -lpthread $ZNG/lib/libz.a"
```
**Expected watch-out:** mingw emits `pigz.exe`, so the Makefile's `ln -f pigz unpigz` step fails with
`ln: failed to access 'pigz'`. That is NOT a build failure — the link already produced `pigz.exe`.
Finish with:
```sh
cp pigz.exe unpigz.exe
```
Tip for a clean rebuild after editing only pigz.c (skips zopfli recompile):
```sh
rm -f pigz.o pigz.exe unpigz.exe
x86_64-w64-mingw32-gcc -O3 -Wall -Wextra -Wno-unknown-pragmas -Wcast-qual -I$ZNG/include -c -o pigz.o pigz.c
x86_64-w64-mingw32-gcc -static -static-libgcc -L$ZNG/lib -o pigz.exe \
  pigz.o yarn.o try.o deflate.o blocksplitter.o tree.o lz77.o cache.o hash.o util.o squeeze.o katajainen.o symbols.o \
  -lm -lpthread $ZNG/lib/libz.a
cp pigz.exe unpigz.exe
```

## Static-link check (do this here in Debian13)
```sh
x86_64-w64-mingw32-objdump -p pigz.exe | grep 'DLL Name'   # ONLY KERNEL32.dll + msvcrt.dll
strings pigz.exe | grep -m1 'pigz 2.'                       # confirms version baked in
```
If you see `libz`, `libwinpthread-1`, or `libgcc_s_seh-1` → static link failed; re-check `-static`
and the absolute `libz.a` path.

## Verify (Phase 5 — WINDOWS-SIDE; cannot run .exe from this WSL instance, no wine)
Hand the user PowerShell commands in `d:\WSL\script`. Key checks: `--version` → "pigz 2.9";
small + >4 GB round-trip with `--progress`; `unpigz` name auto-decompresses (Patch A); hashes match;
`-p 16 -9 -c big.bin > NUL` (use `cmd /c "... > NUL"` in PowerShell) does NOT hang at 0% CPU.

## Stage / Deploy (Phase 6)
- Stage for review WITHOUT touching the live binary — version-named copies:
  ```sh
  cp pigz.exe   /mnt/d/WSL/script/pigz-2.9.exe
  cp unpigz.exe /mnt/d/WSL/script/unpigz-2.9.exe
  ```
- Promote to live ONLY after the user confirms Phase 5 passed. Back up the current ones first
  (leave the existing `*.x86-32bit.bak` alone):
  ```sh
  cd /mnt/d/WSL/script
  cp pigz.exe pigz.exe.v2.8-zlib.bak; cp unpigz.exe unpigz.exe.v2.8-zlib.bak
  cp pigz-2.9.exe pigz.exe; cp unpigz-2.9.exe unpigz.exe
  ```
  Deploy replaces binaries that `wsl-backup.ps1` uses for a ~100 GB VHDX export — hard to reverse.
  Always verify before promoting, and confirm with the user.

## Notes
- The "lock hang" is toolchain-dependent: mingw **14-posix** carries the fixed winpthreads (cures the
  ~0% CPU hang). It does NOT cure a second winpthreads bug — mutex ownership mis-tracking under heavy
  lock churn, which aborts with `already unlocked (...:mutex_unlock)` / `internal threads error`
  (reproducible compressing a highly-compressible >100 GB input, blocks finishing near-instantly).
- **Patch C is now APPLIED** (`yarn.c`, v2.9.1): an `#ifdef _WIN32` lock backend using `SRWLOCK` +
  `CONDITION_VARIABLE` (no owner bookkeeping → `Release` never returns EPERM; `SleepConditionVariableSRW`
  is race-free). Threads still use pthread. Guardrail kept: broadcast while holding the lock, then
  release. Verify imports after build: `objdump -p pigz.exe | grep -iE 'SRWLock|ConditionVariable'`.
  Diagnostic on any recurrence: ~0% CPU = CV/lock wait.
- Full background and rationale: `docs/build-plan.md`; running status: `docs/HANDOFF.md`.
