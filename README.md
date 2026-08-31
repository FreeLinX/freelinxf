FreeLinX will be a hybrid kernel made with Linux Kernel with BSD applications

~ 2026 Kanan Majidzada & Denis Gulmemmedov


FreeLinX is a minimal Linux-based operating system project
designed around a POSIX-oriented userspace, musl libc,
BSD-derived utilities, LLVM/Clang, and runit supervision.

This repository contains the project's:

- Archt
- Spec
- roadmap
- design decisions
- documentation (doodle of doc)
- build strategy

# Building FreeLinX from scratch

This walks through cloning every FreeLinX repo on a fresh machine, building
the toolchain, kernel, and ports, staging a rootfs, and boot-testing the
result in QEMU. It assumes a Linux host (WSL2 is fine) with `clang`, `lld`,
`cmake`, `ninja-build`, `bmake`, `qemu-system-x86_64`, and standard build
tools already available for *host* bootstrap use only (GNU tools may be used
on the host during early bootstrap; the target userland never depends on
them — see each repo's own README for the full rationale).

## Repository layout

Everything assumes sibling directories under one root. Every repo's
`config/default.conf` defaults to `../toolchain`, `../kernel`, `../ports`,
`../src` — so clone them side by side, not nested:

```
~/freelinix/
├── toolchain/
├── kernel/
├── ports/
├── src/
└── iso/
```

## 1. Clone everything

```bash
mkdir -p ~/freelinix
cd ~/freelinix
git clone https://github.com/FreeLinX/toolchain.git
git clone https://github.com/FreeLinX/kernel.git
git clone https://github.com/FreeLinX/ports.git
git clone https://github.com/FreeLinX/src.git
git clone https://github.com/FreeLinX/iso.git
```

## 2. Build the toolchain (blocks everything else)

See `toolchain/BUILD.md` for the full musl sysroot + libc++ cross-build
steps. In short: build musl as a sysroot targeting `x86_64-linux-musl`
using clang + `llvm-ar`/`llvm-ranlib` (not GNU cross-prefixed tools), then
cross-build LLVM's `libcxx`/`libcxxabi`/`libunwind` against that sysroot
with CMake + Ninja.

Verify the result before moving on — this must succeed and print output:

```bash
cat > /tmp/hello.c << 'EOF'
#include <stdio.h>
int main(void) { printf("freelinix alive\n"); return 0; }
EOF

~/freelinix/toolchain/bin/clang \
    --target=x86_64-linux-musl \
    --sysroot=$HOME/freelinix/toolchain/x86_64-linux-musl \
    -fuse-ld=lld -static \
    -o /tmp/hello /tmp/hello.c

/tmp/hello   # must print: freelinix alive
```

## 3. Build the kernel

Independent of `ports`/`src` — only needs the toolchain from step 2.

```bash
export PATH=~/freelinix/toolchain/bin:$PATH
cd ~/freelinix/kernel
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.6.21.tar.xz
tar xf linux-6.6.21.tar.xz
cd linux-6.6.21
cp ../kernel.config .config

make LLVM=1 LLVM_IAS=1 ARCH=x86_64 CC=clang olddefconfig
make LLVM=1 LLVM_IAS=1 ARCH=x86_64 CC=clang -j$(nproc)
```

Produces `arch/x86/boot/bzImage`. See `kernel/README.md` and `SOURCE.md` for
config deviations from stock defconfig.

## 4. Build ports

```bash
cd ~/freelinix/ports
make check                        # confirms the toolchain is detected
make build                        # builds every port
```

To build just one port:

```bash
make build PORT=shells/netbsd-sh
make build PORT=base/cat
make build PORT=sysutils/runit
```

## 5. Stage everything into the rootfs

```bash
cd ~/freelinix/src
make check     # confirms toolchain, kernel, and ports are all detected
make rootfs    # copies rootfs/ template into build/x86_64/rootfs
```

Then pull each port's built output into the tracked `src/rootfs/` template.

**Single-binary ports** (the common case — see "Adding a new port" below):

```bash
cd ~/freelinix/ports
make install PORT=shells/netbsd-sh INSTALL_FLAGS=-r
```

**Multi-binary ports** (suites like `sysutils/runit` that produce more than
one binary FreeLinX needs) must be installed directly against the port
directory, not through the top-level command:

```bash
cd ~/freelinix/ports/sysutils/runit
make install           # stages runsvdir, runsv, sv, chpst into ports/staging/
make install-rootfs    # copies them into src/rootfs/
```

Commit the result in `src`:

```bash
cd ~/freelinix/src
git status
git add rootfs/...
git commit -m "Stage <port> output"
git push
```

## 6. Package an initramfs and boot-test in QEMU

```bash
cd ~/freelinix/src
./scripts/rootfs.sh          # re-stage after any rootfs change

cd build/x86_64/rootfs
find . | cpio -o -H newc | gzip -9 > ~/freelinix/initramfs.cpio.gz
cd ~/freelinix

qemu-system-x86_64 \
    -kernel ~/freelinix/kernel/linux-6.6.21/arch/x86/boot/bzImage \
    -initrd ~/freelinix/initramfs.cpio.gz \
    -append "console=ttyS0 rdinit=/init" \
    -nographic -m 512M
```

`rdinit=/init` runs the real `src/rootfs/init` script: it mounts
proc/sys/dev, then execs `/sbin/runsvdir -P /var/service` if runit is
present (falling back to a rescue shell otherwise). To skip straight to an
interactive shell instead of the full init/runit path — useful for quick
manual testing — use `rdinit=/bin/sh` instead.

Exit QEMU with `Ctrl-A` then `X`.

---

## Adding a new port ("just pull and build")

Most ports are **single-binary**: one upstream program, one compiled
output, staged to one path in the rootfs. This is the easy, fully-automated
case — no manual steps beyond what's below. Use `sysutils/runit` only as a
reference for the rarer multi-binary case (see its section in
`ports/README.md`).

### 1. Create the port directory

```bash
mkdir -p ~/freelinix/ports/<category>/<name>/patches
```

Pick a `<category>` that matches the existing layout (`base`, `shells`,
`sysutils`, etc.) or introduce a new one if nothing fits.

### 2. Write `distinfo`

Names the upstream archive and pins its checksum — never invented, always
computed from the real download:

```
DISTINFO_NAME=<name>-<version>
DISTINFO_ARCHIVE=<name>-<version>.tar.gz
DISTINFO_URL=https://example.org/<name>-<version>.tar.gz
DISTINFO_SHA256=TODO
```

Fetch it once to get the real checksum, then fill in `DISTINFO_SHA256`:

```bash
cd ~/freelinix/ports
./scripts/fetch.sh <category>/<name>
sha256sum dist/<name>-<version>.tar.gz
```

### 3. Write the `Makefile`

For a simple, single-file-style utility, `mk/base-port.mk` does almost
everything — the port just declares what to extract and compile:

```makefile
NAME    := <name>
VERSION := <version>
CATEGORY:= <category>
LICENSE := <license>
TARGET  := $(FREELINX_TRIPLE)
PREFIX  := /

INSTALL_BIN     := <name>
INSTALL_RELPATH := bin/<name>

include ../../mk/base-port.mk

NETBSD_MEMBERS = usr/src/bin/<name>/<name>.c
PORT_SRCS      = <name>.c
```

For anything with its own build system or multiple source files (closer to
`shells/netbsd-sh`), include `mk/port.mk` directly instead and write your
own `do-build` recipe — see `shells/netbsd-sh/Makefile` as the template.

**Set `INSTALL_BIN`/`INSTALL_RELPATH` before the `include` line** — `make`
freezes rule-target names at parse time, so setting them after the include
would not reach the staging rule.

### 4. Build, install, and verify

```bash
cd ~/freelinix/ports
make build PORT=<category>/<name>
make install PORT=<category>/<name>

file staging/<INSTALL_RELPATH>
```

Confirm it's a genuine static musl binary, not accidentally linked against
the host's glibc:

```bash
ldd staging/<INSTALL_RELPATH>   # expect: not a dynamic executable
```

### 5. Push it into `src`'s rootfs

```bash
make install PORT=<category>/<name> INSTALL_FLAGS=-r
cd ~/freelinix/src
git status
git add rootfs/<INSTALL_RELPATH>
git commit -m "Stage <name> from FreeLinX/ports"
git push
```

That's the whole loop for a normal port: `distinfo` → `Makefile` →
`make build` → `make install ... -r` → commit in `src`. No manual staging
steps, no per-port hand-written install targets — those are only needed for
multi-binary suites like runit.

[IMPORTANT] This is not the OS source repository. Just a doodle.
