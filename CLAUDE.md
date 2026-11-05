# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of independent example/reference file systems built on [macFUSE / OSXFUSE](https://osxfuse.github.io/) (formerly MacFUSE), grouped by implementation language. It is a fork of `kfinteractive/filesystems`; there is no top-level build, no test suite, and no shared dependency between the subprojects (except `unixfs`, below). Each subproject builds on its own.

**These only build and run on macOS** with macFUSE installed (headers in `/usr/local/include/osxfuse`, libs in `/usr/local/lib`, `OSXFUSE.framework` in `/Library/Frameworks`). They will not compile in the Linux cloud container, so don't try to verify changes by building here — review by reading, and say so when a change is unverified.

## Layout and build commands

- `filesystems-c/` — C/C++ against the libfuse API (`-losxfuse`). Each dir has its own `Makefile`:
  - `hello/` (`hello`, high-level API, and `hello_ll`, low-level API), `loopback/`, `clock/` (`clock_ll`, low-level), `verybigfs/`: `cd filesystems-c/<dir> && make` (`make clean` to remove). The Makefiles share a pattern: `OSXFUSE_ROOT` (default `/usr/local`, alt `/opt/local`) sets include/lib paths, and `FUSE_USE_VERSION` / `-D_DARWIN_USE_64_BIT_INODE` are set via `CFLAGS_OSXFUSE`.
  - `loopback/`: `make` prints an example mount command (`sudo ./loopback /Volumes/loop -omodules=threadid:subdir,subdir=/tmp/dir -oallow_other,native_xattr,volname=LoopbackFS`).
  - `unixfs/`: `make` here loops over `ancientfs minixfs sysvfs ufs`; each subdir builds its own binary and compiles with `-Wall -Werror`.
  - `procfs/`: C++, needs `pcre`/`pcrecpp` (`pcre-config`) and old Carbon/QuickTime/i386 flags (`-arch i386`, `MacOSX10.6.sdk`) — likely needs Makefile changes to build on modern macOS. Run foregrounded (`sudo ./procfs /proc -f`) or via `procfs.plist`/launchd. `make -C sequencegrab` builds the static lib it links.
  - `grabfs/`: C++ plus a Cocoa app in `Cocoa/WindowFS.xcodeproj`; has a `Makefile`.
- `filesystems-objc/` — Objective-C using `GMUserFileSystem` from `OSXFUSE.framework`. Each of `HelloFS`, `LoopbackFS`, `YTFS`, `AccessibilityFS`, `SpotlightFS` is an Xcode project (`xcodebuild -project <Name>.xcodeproj`). Shared helpers (`NSError+POSIX`, `NSImage+IconData`, `YTVideo`) live in `filesystems-objc/Support/` and are referenced from the projects, not copied.
  - `LoopbackFS` can also build as a CLI: `gcc -o loop ../Support/NSError+POSIX.m LoopbackFS.m loop.m -I../Support -framework OSXFUSE -framework Foundation` (its README still says `MacFUSE`).
- `filesystems-swift/LoopbackFS/` — Swift port of LoopbackFS; Xcode project, deployment target 10.12, imports `<OSXFUSE/OSXFUSE.h>` via `Bridging-Header.h`.
- `support/testing/posix_compat_test/` — C++ POSIX-semantics test harness (`make` produces `posix_compat_test`), meant to be pointed at a mounted file system. `support/debugging/` holds a DTrace script (`macfuse_u_fuse_fs.d`).

There is no lint config and no automated test runner. The "tests" are mounting the file system and exercising it (or `posix_compat_test`).

## Architecture notes (things that span multiple files)

- **Two FUSE API styles.** C examples use libfuse directly: high-level path-based (`hello.c`, `loopback.c`, `procfs`, `unixfs`) vs low-level inode-based (`hello_ll.c`, `clock_ll.c`). ObjC/Swift examples instead subclass/delegate through `GMUserFileSystem` (path-based, in `OSXFUSE.framework`), with a controller (`*Controller.m`/`AppDelegate.swift`) that mounts it and observes `GMUserFileSystemDidMount`/`DidUnmount`/`MountFailed` notifications.
- **`unixfs` is the only multi-file shared design.** `filesystems-c/unixfs/common/unixfs/` is a generic FUSE layer (`unixfs.c`, `unixfs_internal.c`) that each concrete file system (`ancientfs`, `minixfs`, `sysvfs`, `ufs`) plugs into via its `unixfs_*.c` adapter; `common/linux/` supplies a small Linux-kernel-compat shim (`linux.c`, `kernel/` headers and `lib/parser.c`) so Linux-origin filesystem code can be reused. Editing `common/` affects all four binaries, and `make clean` in each subdir also deletes the `.o`/`.d` files in `common/`, so build/clean them as a set. Dependency tracking uses generated `.d` files via the `%.o` rule.
- **Version-conditional code.** `loopback.c` gates features on `FUSE_VERSION` (>= 29) and `MAC_OS_X_VERSION_MIN_REQUIRED/MAX_ALLOWED` (e.g. `renamex_np` with `RENAME_SWAP` for macOS 10.12+/10.15 support); preserve these guards when changing it. Recent upstream commits are macOS-compat fixes (hardened runtime / code signing / entitlements for `LoopbackFS.app`).
- **Mixed licensing/provenance.** Subprojects come from different authors (Amit Singh's `procfs`/`unixfs`, Google MacFUSE examples, KF Interactive's Swift LoopbackFS) and carry their own `COPYING`/`LICENSE` files; keep them intact.

## Session notes

- The working branch is `claude/init-2r0831`; the upstream base branch is `master`.
- A sibling checkout, `/home/user/nextjs-preview-mode` (`Devjeverett/nextjs-preview-mode`), is attached to the same session but is currently an empty repo with no commits, so there is nothing in it to document yet.
