# static-gdb-arm64

Statically-linked [GDB](https://www.gnu.org/software/gdb/) that runs **natively on
ARM64 (AArch64) Linux machines** — single portable binaries with no shared-library
dependencies, for copying onto servers, SBCs (Raspberry Pi 4/5, Rockchip, ...),
containers, or any arm64 box where you can't install packages.

Two variants:

| Binary | ~Size | Description |
|---|---|---|
| `gdb-<ver>-arm64-static-nopython` | ~14 MB | TUI included, no Python scripting |
| `gdb-<ver>-arm64-static-python` | ~20 MB | Embedded CPython 3.12 for GDB Python scripting |

Target profile: **AARCH64 (ARMv8-A, LP64), little-endian, glibc 2.40** (needs a
reasonably modern kernel on the device). Big-endian `aarch64_be` is not covered.
Check a device with `readelf -h /bin/sh | grep -E 'ELF|Machine'`.

## Quick start

Requirements: Ubuntu/Debian-ish **x86_64** host (the build cross-compiles), with
`build-essential`, `curl`, and:

```bash
sudo apt-get install -y \
    libgmp-dev libmpfr-dev libexpat1-dev zlib1g-dev libncurses-dev texinfo \
    gcc-14-aarch64-linux-gnu g++-14-aarch64-linux-gnu \
    libc6-dev-arm64-cross binutils-aarch64-linux-gnu \
    qemu-user-static
```

Then:

```bash
./build.sh
```

Binaries land in `dist/`, full install trees under `install/<triplet>-<variant>/`.
A full build takes roughly 15–25 minutes on a 20-core machine. The script is
re-runnable: sources, host Python, cross libraries, and cross Pythons are
skipped when already present.

### Options

```bash
GDB_VERSION=16.3     PYTHON_VERSION=3.12.7    # upstream versions
TRIPLETS="aarch64-linux-gnu"                  # triplet to build
VARIANTS="nopython python"                    # subset to build
CROSS_GCC_VER=14    # cross gcc package series on the host
JOBS=$(nproc)       # parallel jobs
BASE=/path/to/dir   # where to place src/deps/build/install/dist (default: repo dir)
```

## Deploying to a device

```bash
scp dist/gdb-16.3-arm64-static-nopython user@device:/usr/local/bin/gdb
```

For the **python** variant also copy the data-directory (its path is baked in
at build time; keep the relative layout or pass `--data-directory`):

```bash
scp -r install/aarch64-linux-gnu-python/share user@device:/usr/local/
```

## Verifying on an x86_64 host (qemu)

The build script sanity-checks every binary under qemu-user. You can also run a
full end-to-end debug session on your workstation:

```bash
# target program under qemu's gdbserver stub
qemu-aarch64-static -g 12345 ./my-arm64-binary &
qemu-aarch64-static ./dist/gdb-16.3-arm64-static-nopython \
    -batch -ex 'file ./my-arm64-binary' -ex 'target remote :12345' \
    -ex 'break main' -ex continue -ex 'info registers pc'
```

## Caveats

- **Thread debugging is limited**: a glibc-static executable cannot `dlopen`
  `libthread_db` (same limitation as any statically linked GDB).
- **Python variant**: extension modules relying on `dlopen` cannot load
  (`import ctypes` fails); pure-Python GDB scripts work. pwndbg/gef will not
  work with a static build.
- Native target only: this debugs arm64 little-endian userland. To cross-debug
  arm64 from x86_64 instead, a `--enable-targets=all` x86_64 build is a better
  fit.
- Static glibc 2.40 binaries need a kernel of roughly 3.2 or newer on the device.

## How it works / notable gotchas

1. **Cross toolchain**: Ubuntu's `gcc-14-aarch64-linux-gnu` +
   `libc6-dev-arm64-cross` packages (static glibc archives included). GDB's
   support libraries (GMP/MPFR — hard requirements of GDB 16 — plus expat,
   zlib, ncurses) are not packaged for cross use, so they are cross-built
   statically from source.
2. **libtool swallows `-static`**: GDB's final link runs through libtool, which
   consumes a plain `-static` flag and silently produces a dynamic binary. The
   build relinks with `make -C gdb LDFLAGS="-all-static"`.
3. **qemu-user as the Python bridge**: GDB's configure must *execute* the
   Python it links against. The cross-built Python (static libpython, dynamic
   glibc) is wrapped in a `qemu-aarch64-static -L /usr/aarch64-linux-gnu`
   script passed to `--with-python=` — configure runs the target interpreter
   under emulation and link-tests the real cross `libpython3.12.a`.
4. **CPython 3.12 cross quirks**: `--with-build-python=<host python>` is
   required; `ac_cv_buggy_getaddrinfo=no` (a runtime test false-fails when
   cross-compiling); out-of-tree builds require a pristine source tree, which
   is why every Python build here (host and cross) is out-of-tree.
5. **ncurses widec**: cross ncurses installs only the widec flavor
   (`libncursesw.a`), while readline/TUI look for `-lncurses`/`-ltinfo`; the
   build provides symlink aliases (ABI-compatible for readline's use).

Build layout (all gitignored): `src/`, `deps/` (host + cross Pythons and
libraries), `build-*/`, `install/`, `dist/`.

## License

GDB is GPLv3; statically-linked GDB binaries are derivative works, so if you
distribute the binaries you must offer the corresponding source (this repo plus
the pinned upstream versions is a reasonable way to do that). No license is
claimed over these build files — add one of your choosing before publishing.
