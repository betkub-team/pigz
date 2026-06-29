# pigz rebuild — HANDOFF (work paused mid Phase 3)

Source lives in **Debian13** under `/home/admin/projects/pigz` (pigz v2.8) and
`/home/admin/projects/zlib-ng` (zlib-ng 2.2.4). Full plan: `docs/build-plan.md`.
Goal: cross-compile a **static Windows `pigz.exe`/`unpigz.exe`** linked against zlib-ng (runtime
AVX2), with a `--progress` meter and the winpthreads hang avoided via the modern toolchain.

All edits/builds run from inside Debian13:
`wsl -d Debian13 -u admin -- bash -lc '...'`. Editing pigz.c from Windows works via the UNC path
`\\wsl.localhost\Debian13\home\admin\projects\pigz\pigz.c`.

## DONE
- **Phase 0** — toolchain installed: `gcc-mingw-w64-x86-64` (GCC **14-posix**, fixed winpthreads) +
  `cmake 3.31.6`. The mingw alternative is set to the **posix** variant:
  `sudo update-alternatives --set x86_64-w64-mingw32-gcc /usr/bin/x86_64-w64-mingw32-gcc-posix`.
- **Phase 1** — cloned pigz `v2.8` and zlib-ng `2.2.4` (shallow) into `/home/admin/projects`.
- **Phase 2** — zlib-ng cross-built static, zlib-compat. Artifacts:
  `/home/admin/projects/zlib-ng/install-win/lib/libz.a` + `include/{zlib.h,zconf.h}` (COFF archive).
  NOTE: install used an explicit `--prefix` because `$PWD` inside the `wsl.exe` call resolved to the
  Windows CWD (`/mnt/d/WSL/script`) — always pass an **absolute** `--prefix`, not `$PWD`.

## STATUS UPDATE (2026-06-29): Phase 3 COMPLETE, Phase 4 COMPLETE.
- Phase 3 remaining patches 7, 8, 9 all applied (file-branch `in_size`/`in_seen`,
  final `show_progress(1)` newline, `--progress` help line). Native `gcc -fsyntax-only` clean.
- Phase 4 cross-build succeeded: `pigz.exe` (732018 bytes) built with mingw 14-posix + zlib-ng.
  The Makefile `ln -f pigz unpigz` step errors (mingw emits `pigz.exe`, not `pigz`) — handled by
  `cp pigz.exe unpigz.exe` as the watch-out predicted. Both `.exe` now exist in the repo dir.
- Static link confirmed: `objdump -p pigz.exe` shows only KERNEL32.dll + msvcrt.dll
  (no libz/libwinpthread/libgcc_s).
- **BLOCKED on Phase 5**: runtime verification needs Windows (no wine in this WSL instance).
  Deploy target `/mnt/d/WSL/script/` IS reachable (current pigz.exe = 263680 bytes, plus
  `*.x86-32bit.bak`). Do NOT deploy (Phase 6) until Phase 5 passes Windows-side.
- **Version bumped to 2.9** (`#define VERSION "pigz 2.9"` + header + changelog). Staged copies in
  `/mnt/d/WSL/script/` are version-named: `pigz-2.9.exe` / `unpigz-2.9.exe` (732018 bytes each).
- **Phase 7 DONE**: created `.claude/skills/build-pigz/SKILL.md` (full recipe), project `CLAUDE.md`,
  and updated `d:\WSL\script\CLAUDE.md` (v2.9 / zlib-ng / --progress / source location). `/init`
  effectively satisfied by the hand-written CLAUDE.md.
- **Phase 5 PASSED** (user verified Windows-side) → **Phase 6 DEPLOYED**: backed up live binaries as
  `pigz.exe.v2.8-zlib.bak` / `unpigz.exe.v2.8-zlib.bak`, promoted v2.9 to live. md5 of live
  `pigz.exe`/`unpigz.exe` == `pigz-2.9.exe` (da50a9b8…). `*.x86-32bit.bak` left untouched.
- **Phase 8 DONE**: wired `--progress` into the PowerShell scripts in `d:\WSL\script`.
  - `wsl-backup.ps1` (compress block ~688): removed the output-size estimation poll loop; pigz now
    runs with `--progress` and writes its accurate meter (input-based %, GB/GB, MB/s) to stderr via
    an inherited console handle (`UseShellExecute=$false`, stderr NOT redirected). `$expectedRatio`
    (line 51) marked DEPRECATED/unused but kept for rollback. `$stopwatch`/`$gzInProgress` still used.
  - `restore.ps1` (decompress ~491): added `--progress` to `pigz -d`, plus a `Write-Host ""` to close
    the line.
  - Not syntax-checked (no pwsh in this WSL instance) — eyeball-verified; confirm Windows-side.
- **ALL PHASES COMPLETE.** Rollback if needed: copy `*.v2.8-zlib.bak` back over `pigz.exe`/`unpigz.exe`
  and revert the two `.ps1` edits.

## (HISTORICAL) IN PROGRESS — Phase 3 (patching `pigz.c`). Edits ALREADY APPLIED:
1. **struct g** — added fields after `out_check`: `length_t in_size; length_t in_seen; int progress;`.
2. **show_progress(int final)** — new helper inserted right after `grumble()`. Throttled (~250 ms)
   `\r` line to **stderr**; uses `g.in_seen` / `g.in_size`; shows "compressing/decompressing", %,
   GB/GB, MB/s; falls back to bytes+rate when size unknown.
3. **readn()** — after the read loop, added:
   `if (g.progress && desc == g.ind) { g.in_seen += got; show_progress(0); }`.
4. **Patch A (program name)** — in `main()`, replaced the `strrchr(argv[0], '/')` block with a scan for
   both `/` and `\`, plus a `#ifdef _WIN32` block that strips a trailing `.exe` (any case) into a static
   `progbuf[260]`. This makes `unpigz.exe` auto-decompress.
5. **--progress option** — in `option()`, at the top of the long-option branch (right after `arg++;`),
   added explicit handling for `--progress` (sets `g.progress=1`) and `--no-progress` (clears it),
   returning 1. No short letter used.
6. **process() stdin branch** — after `len = 0;` added `g.in_size = 0; g.in_seen = 0;`.

## REMAINING — Phase 3 (DO THESE NEXT):
7. **process() file branch** — after the input is opened:
   ```c
   g.ind = open(g.inf, O_RDONLY, 0);
   if (g.ind < 0)
       throw(errno, "read error on %s (%s)", g.inf, strerror(errno));
   ```
   add:
   ```c
   g.in_size = S_ISREG(st.st_mode) ? (length_t)st.st_size : 0;
   g.in_seen = 0;
   ```
   (`st` still holds the earlier `lstat` result here — correct size for both compress=uncompressed
   file and decompress=.gz file.)
8. **Final progress newline** — near the end of `process()`, just after the compress/decompress block
   (the existing `if (g.verbosity > 1) { putc('\n', stderr); fflush(stderr); }`), add:
   ```c
   if (g.progress) {
       show_progress(1);
       putc('\n', stderr);
       fflush(stderr);
   }
   ```
9. *(optional, polish)* add a `--progress` line to `help()` text.
10. *(optional, Patch C)* native Win32 yarn backend (`SRWLOCK`/`CONDITION_VARIABLE`) in `yarn.c`.
    Decided OPTIONAL — GCC 14 posix already carries the fixed winpthreads, which is the primary hang
    fix. Only add if a hang is later observed (diagnostic: 0% CPU = lock wait).

## Phase 4 — build (after patches compile-check)
```sh
cd /home/admin/projects/pigz
ZNG=/home/admin/projects/zlib-ng/install-win
make CC=x86_64-w64-mingw32-gcc \
  CFLAGS="-O3 -Wall -Wextra -Wno-unknown-pragmas -Wcast-qual -I$ZNG/include" \
  LDFLAGS="-static -static-libgcc -L$ZNG/lib" \
  LIBS="-lm -lpthread $ZNG/lib/libz.a"
```
Watch-outs:
- The Makefile target is `pigz`; mingw may emit `pigz` or `pigz.exe`. If you get `pigz` (no ext),
  rename to `pigz.exe`. Then `cp pigz.exe unpigz.exe`.
- If link fails on pthreads, confirm the posix variant is active (Phase 0 update-alternatives).
- `make` also builds bundled zopfli objects — expected.

## Phase 5 — verify (Windows side; this session's Bash/PowerShell run .exe directly)
Copy both .exe to `d:\WSL\script` then:
- `./pigz.exe --version` → "pigz 2.8".
- `objdump -p pigz.exe | grep 'DLL Name'` → only KERNEL32/ucrtbase etc. (no libz/libwinpthread/libgcc_s).
- ≥4 GB round trip with `--progress`: `pigz.exe --progress -k big.bin`, `pigz.exe -t big.bin.gz`,
  `unpigz.exe --progress -k big.bin.gz` (proves Patch A), compare hashes.
- `pigz.exe -p 16 -9 -c big.bin > NUL` thread stress — must not hang at 0% CPU.

## Phase 6 — deploy
Back up current repo binaries → `pigz.exe.v2.8-zlib.bak` / `unpigz.exe.v2.8-zlib.bak` (leave the
existing `*.x86-32bit.bak` alone), then copy the new `.exe` into `d:\WSL\script\`.

## Phase 7 — project setup (user asks)
- `/init` the project at `/home/admin/projects/pigz` (generate CLAUDE.md).
- Create skill `/home/admin/projects/pigz/.claude/skills/build-pigz/SKILL.md` encoding Phases 0–6.
- This plan already moved to `docs/build-plan.md` (done). Update `d:\WSL\script\CLAUDE.md` to note the
  new zlib-ng build, `--progress`, and that source/skill live here.

## Phase 8 (optional)
Wire `--progress` into `wsl-backup.ps1` / `restore.ps1` instead of the file-size polling estimate.
