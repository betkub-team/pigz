# Rebuild pigz for Windows 11 — faster (zlib-ng), no lock-hang, with progress output
### (source in WSL Debian13, cross-compiled to a Windows .exe)

## Context

The repo `d:\WSL\script` ships prebuilt `pigz.exe` / `unpigz.exe` (v2.8, 64-bit) used by
`wsl-backup.ps1` to compress the WSL `Debian13` VHDX export (~100 GB). There is **no C source in the
repo** — only binaries and a legacy 32-bit `*.x86-32bit.bak` pair. The user wants the pigz binaries
themselves improved for Windows 11, with the **source developed inside WSL Debian13 at
`/home/admin/projects`** and **cross-compiled back to a Windows `.exe`** (the backup scripts call
`pigz.exe`, so the artifact must stay a Windows binary).

Four goals, confirmed:
1. **Faster** compress/decompress on modern Win11 CPUs → link pigz against **zlib-ng** (SIMD/AVX2 deflate
   with runtime dispatch) instead of stock zlib.
2. **Fix "hanging waiting on lock"** seen in pigz GitHub issues.
3. **Add live progress output** during compression *and* decompression.
4. Preserve existing behavior: `unpigz.exe` auto-decompresses; ≥4 GB files work.

Plus two setup asks: create a **`build-pigz` project skill** so future sessions can rebuild easily, run
**`/init`** for the project, and **move this plan** into the project.

### Environment (verified in Debian13)
- Debian13 running; user `admin`; `/home/admin/projects` exists.
- Present: `git 2.47.3`, `make 4.4.1`, `gcc 14.2.0`. **Missing but apt-installable:** `cmake`, and the
  mingw cross toolchain `gcc-mingw-w64-x86-64` (Candidate **14.2.0** — modern, ships the **fixed
  winpthreads**, which is the primary cure for goal #2).

### Key research findings (drive the approach)
- **zlib-ng does runtime CPU dispatch** — built with defaults (`WITH_RUNTIME_CPU_DETECTION=ON`,
  `WITH_AVX2/SSE2=ON`) it picks the best SIMD path via `cpuid` at runtime. One build = portable *and*
  AVX2-fast. **Never** use `-march=native` / `WITH_NATIVE_INSTRUCTIONS` (illegal-instruction crashes on
  older CPUs). `-march` on pigz itself is irrelevant; all SIMD lives in zlib-ng.
- **The "lock hang" is not a pigz core bug.** pigz's `yarn.c` `twist_()` broadcasts the condition variable
  *under* the mutex and `wait_for_()` re-checks in a loop — the deadlock-safe pattern. The real cause is a
  **winpthreads (mingw) CV race** (mingw bugs #774/#710), **fixed 2022-12-29**. Old toolchains under
  sustained broadcasting (every 128 KB job over 100 GB) park all workers in `pthread_cond_wait` → 0% CPU,
  no progress = "waiting on lock." Debian's mingw **14.2** carries the fix → primary cure. Optional
  hardening: a native Win32 yarn backend that bypasses winpthreads. The `unpigz -l -d` double-init race
  (pigz #96) is already fixed in 2.8 — just regression-check it.

## Recommended approach

Develop the source in `Debian13:/home/admin/projects`, cross-compile a **static** `pigz.exe` from
**pigz v2.8** + **zlib-ng (zlib-compat, static)** using Debian's **mingw-w64 14.2** cross toolchain, with
three source patches, then drop the new binaries into `d:\WSL\script` (old ones kept as `.bak`).

All build commands run inside Debian13: `wsl -d Debian13 -u admin -- bash -lc '...'`.

### Phase 0 — Toolchain (apt; one-time)
```sh
sudo apt update
sudo apt install -y gcc-mingw-w64-x86-64 cmake   # git/make/gcc already present
# Use the POSIX-threads mingw variant so pigz's pthreads → winpthreads:
sudo update-alternatives --set x86_64-w64-mingw32-gcc /usr/bin/x86_64-w64-mingw32-gcc-posix
x86_64-w64-mingw32-gcc --version   # confirm 14.2.0 (fixed winpthreads)
```

### Phase 1 — Get the source into /home/admin/projects
```sh
cd /home/admin/projects
git clone --branch v2.8  --depth 1 https://github.com/madler/pigz.git
git clone --branch 2.2.4 --depth 1 https://github.com/zlib-ng/zlib-ng.git
```

### Phase 2 — Cross-build zlib-ng as a static Windows `libz.a`
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
  -DCMAKE_INSTALL_PREFIX="$PWD/install-win"
cmake --build build-win -j && cmake --install build-win
# → install-win/lib/libz.a + install-win/include/{zlib.h,zconf.h}
```

### Phase 3 — Patch pigz (`pigz.c`, optionally `yarn.c`)
**Patch A — Windows program-name detection** (so `unpigz.exe` decompresses). In `main()` replace the
`g.prog = strrchr(argv[0], '/')` lines with logic that scans for both `/` and `\` separators and, under
`#ifdef _WIN32`, strips a trailing `.exe` (case-insensitive) into a static `MAX_PATH` buffer. Confirm the
exact lines: `grep -n 'g.prog = strrchr' pigz.c`. (Re-derives the patch the current binary already has.)

**Patch B — Live progress meter to stderr** (goal #3). Add globals to struct `g`: `length_t in_size;`
(total input size, `0` for stdin/pipe), `int progress;`, plus a last-print timestamp + byte count. Add a
throttled (~250 ms) `show_progress()` helper writing a `\r`-updated line to **stderr only** (stdout must
stay a pure data stream for `-c` pipelines):
  - Compression: set `g.in_size` from the input file's `st_size` when a regular file opens; call
    `show_progress("compressing", g.in_tot, g.in_size)` from `load()` right after `g.in_tot += got;`
    (one counting thread → race-free).
  - Decompression: total output size is unknown → show **bytes processed + MB/s** (optionally % of the
    *compressed* input consumed); call from the inflate loop on the main thread.
  - Gate behind a new `--progress` long option (add to the parser); optionally auto-enable when
    `verbosity>=2 && isatty(stderr)`. Print a final newline on completion.

**Patch C (optional hardening) — native Win32 yarn backend.** `yarn.c` abstracts mutex+cond; add an
`#ifdef _WIN32` backend using `SRWLOCK` + `CONDITION_VARIABLE` (`AcquireSRWLockExclusive`,
`SleepConditionVariableSRW`, `WakeAllConditionVariable`) to bypass winpthreads entirely. **Guardrail:**
keep the broadcast happening while the lock is held (never signal after unlock). If fiddly, the mingw 14.2
toolchain alone is expected to fix the hang — treat C as belt-and-suspenders.

Regression-check the #96 fix: `process()` runs list-info **before** the decode `in_init()`/`get_header()`
(true in 2.8).

### Phase 4 — Cross-build pigz.exe (static, no DLL deps)
```sh
cd /home/admin/projects/pigz
ZNG=/home/admin/projects/zlib-ng/install-win
make CC=x86_64-w64-mingw32-gcc \
  CFLAGS="-O3 -Wall -Wextra -Wno-unknown-pragmas -Wcast-qual -I$ZNG/include" \
  LDFLAGS="-static -static-libgcc -L$ZNG/lib" \
  LIBS="-lm -lpthread $ZNG/lib/libz.a"      # absolute path forces static zlib-ng
cp pigz.exe unpigz.exe                        # Patch A → auto-decompress by name
```
`-static` pulls winpthreads + libgcc in statically → no `libwinpthread-1.dll` / `libgcc_s_seh-1.dll` /
`zlib1.dll` runtime deps.

### Phase 5 — Verify (Windows-side; must pass before replacing repo binaries)
Copy the two `.exe` to `d:\WSL\script` and run from Windows (this session's Bash/PowerShell run Windows
binaries directly):
```sh
./pigz.exe --version                          # "pigz 2.8"; banner reflects zlib-ng
objdump -p pigz.exe | grep 'DLL Name'         # only KERNEL32/ucrtbase etc — no libz/libwinpthread/libgcc_s
# >=4 GB round-trip (the old 32-bit failure) with live progress:
# (create a >4GB file, then:)
./pigz.exe --progress -k big.bin              # watch progress line
./pigz.exe -t big.bin.gz                       # integrity OK
./unpigz.exe --progress -k big.bin.gz         # proves Patch A + progress on unzip
# compare hashes of original vs decompressed
./pigz.exe -p 16 -9 -c big.bin > NUL          # thread stress — must NOT hang at 0% CPU
```
Optionally benchmark wall-clock vs the current binary on a real export to confirm the zlib-ng speedup.

### Phase 6 — Deploy into the repo
- Back up current binaries: `pigz.exe` → `pigz.exe.v2.8-zlib.bak`, same for `unpigz.exe` (leave the
  existing `*.x86-32bit.bak` untouched).
- Copy the new `pigz.exe` / `unpigz.exe` into `d:\WSL\script\`.

### Phase 7 — Project setup in WSL (the user's extra asks)
- **`/init`** the project at `/home/admin/projects/pigz` → generates a `CLAUDE.md` documenting the source
  + the cross-compile build.
- **Create the `build-pigz` project skill** at
  `/home/admin/projects/pigz/.claude/skills/build-pigz/SKILL.md` — frontmatter
  (`name: build-pigz`, a `description` covering "cross-compile pigz.exe with zlib-ng on Debian mingw-w64")
  plus the full recipe from Phases 0–6 (apt deps, zlib-ng cmake, the three patches, the `make` line, the
  verification + deploy steps) so a future session can `git clone` → rebuild in one shot.
- **Move this plan** into the project: copy this file to
  `/home/admin/projects/pigz/docs/build-plan.md`.
- Update `d:\WSL\script\CLAUDE.md` to record: now linked against zlib-ng (runtime AVX2 dispatch), the
  winpthreads hang fix (mingw 14.2 + optional Win32 yarn backend), the new `--progress` flag, and that the
  source/skill live in `Debian13:/home/admin/projects/pigz`.

### Phase 8 (optional) — wire `--progress` into the PowerShell scripts
`wsl-backup.ps1` currently estimates progress by polling output file size against a hardcoded
`$expectedRatio = 0.32`; with native `--progress` it can surface pigz's own accurate line (and
`restore.ps1` likewise for decompression). Small, isolated change — only if wanted.

## Files created / modified
- **WSL `/home/admin/projects/zlib-ng/`** — cross-built static `libz.a` (not in the Windows repo).
- **WSL `/home/admin/projects/pigz/pigz.c`** — Patch A (argv[0]/`.exe`), Patch B (`--progress`).
- **WSL `/home/admin/projects/pigz/yarn.c`** *(optional)* — Patch C (Win32 yarn backend).
- **WSL `/home/admin/projects/pigz/CLAUDE.md`** — from `/init`.
- **WSL `/home/admin/projects/pigz/.claude/skills/build-pigz/SKILL.md`** — new build skill.
- **WSL `/home/admin/projects/pigz/docs/build-plan.md`** — this plan, relocated.
- **`d:\WSL\script\pigz.exe`, `unpigz.exe`** — replaced (old kept as `.bak`).
- **`d:\WSL\script\CLAUDE.md`** — document the new build + source/skill location.
- *(optional)* **`wsl-backup.ps1` / `restore.ps1`** — adopt `--progress`.

## Risks / notes
- The "lock hang" is **toolchain-dependent**; mingw 14.2 is the main fix, Patch C is optional insurance.
  Diagnostic on any recurrence: ~0% CPU = CV/lock wait (toolchain/Patch C); ~100% CPU = a codepath spin
  (different class, out of scope).
- Use the **POSIX**-threads mingw variant so pigz's `-lpthread` resolves to (fixed) winpthreads; or, if
  Patch C is adopted, the win32-threads variant also works.
- Pin zlib-ng to a release tag (2.2.4) for reproducibility; bumping later just re-runs Phase 2.
- Skip `-march` for a distributable binary; `-flto` is a safe optional extra.
