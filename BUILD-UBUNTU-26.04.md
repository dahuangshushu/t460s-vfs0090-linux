# Building on Ubuntu / Kubuntu 26.04 ("resolute")

This document records how the driver in this repository was built and packaged
on **Kubuntu 26.04 (resolute)**, including the one source-level compatibility
fix that was required, and the reasoning behind it. It is written so that
another T460s owner can reproduce the build on the same release.

---

## 1. Summary

| Item | Value |
| --- | --- |
| Target OS | Ubuntu / Kubuntu 26.04 (codenamed *resolute*) |
| Architecture | amd64 |
| Upstream base | `3v1n0/libfprint-tod-vfs0090`, version `0.8.5` / package `0.96.91~f1` |
| Source change required | **1 line** in `meson.build` |
| Change | `dependency('udev')` → `dependency('libudev')` |
| Build command | `dpkg-buildpackage -b -us -uc` |
| Produced artifact | `libfprint-2-tod-vfs0090_0.96.91~f1_amd64.deb` |
| Published asset name | `libfprint-2-tod-vfs0090_0.96.91.f1_amd64.deb` (see §6.1) |
| Not published | `libfprint-2-tod-vfs0090-dbgsym_0.96.91~f1_amd64.ddeb` (see §6) |

The driver source (`vfs0090.c`, `vfs0090.h`) is **completely unmodified** from
upstream. Only the Meson build definition changed.

---

## 2. Why not the upstream PPA

Upstream publishes Ubuntu packages via:

```
ppa:3v1n0/libfprint-vfs0090
```

That PPA **does not publish a `resolute` series** (the 26.04 codename). Adding
it on 26.04 therefore yields no candidate package for the release — there is
simply nothing to install, regardless of how the PPA is added.

There is no equivalent of "just use the PPA" on 26.04, so building from source
is the supported approach. The `.deb` attached to the GitHub Release was built
by exactly the procedure below.

---

## 3. The `udev` → `libudev` compatibility failure

### Symptom

Meson configuration aborts immediately:

```
Run-time dependency udev found: NO (tried pkgconfig and cmake)
meson.build:15:0: ERROR: Dependency "udev" not found, tried pkgconfig and cmake
```

The build stops before compiling anything. This is a **build-time dependency
resolution** failure, not a compiler error, which is why the fix is in
`meson.build` and not in the C sources.

### Root cause

Upstream declares:

```meson
udev_dep = dependency('udev')
```

`dependency('udev')` asks pkg-config (and then CMake) for a module named
`udev`. Historically, some distributions shipped a `udev.pc` file, so this
resolved. On current Debian/Ubuntu — including 26.04 — the udev development
metadata is packaged by **`libudev-dev`** and the pkg-config file it installs is
named **`libudev.pc`**. There is no `udev.pc`, so the lookup fails.

Confirm this on your own system:

```console
$ pkg-config --exists udev && echo "udev.pc present" || echo "no udev.pc"
no udev.pc

$ pkg-config --exists libudev && echo "libudev.pc present"
libudev.pc present

$ pkg-config --variable=udevdir libudev
/usr/lib/udev
```

The last command is the important one: it confirms `libudev.pc` still exports
the `udevdir` variable that `meson.build` relies on to know where to install the
udev rule.

### The fix

```diff
--- a/meson.build
+++ b/meson.build
@@ -12,7 +12,7 @@
 vfs0090_deps = []
 
 libfprint_tod_dep = dependency('libfprint-2-tod-1')
-udev_dep = dependency('udev')
+udev_dep = dependency('libudev')
 
 vfs009x_deps += libfprint_tod_dep
 vfs009x_deps += dependency('nss')
```

`libudev` and `udev` refer to the **same library**; only the pkg-config module
name differs. Nothing else in the build needed changing:

* `udev_dep.get_pkgconfig_variable('udevdir')` still works — verified above.
* No C source references the dependency name.
* The installed udev rule and its path are unchanged.

This is the entire delta between this snapshot and upstream, and it is what
makes the package buildable on 26.04.

> Note: if you prefer to keep the tree byte-identical to upstream, you can
> instead pass a dependency override at configure time. Editing `meson.build` is
> simpler and is what this snapshot does.

---

## 4. Full build procedure

### Install build dependencies

```bash
sudo apt update
sudo apt install -y \
    build-essential debhelper devscripts meson pkg-config \
    libfprint-2-tod-dev libfprint-2-dev libglib2.0-dev libgusb-dev \
    libpixman-1-dev libssl-dev libnss3-dev libusb-1.0-0-dev \
    libudev-dev udev
```

`libfprint-2-tod-dev` is the one that matters most: it provides
`libfprint-2-tod-1.pc`, which is what lets Meson find the TOD drivers
directory. Without it, `dependency('libfprint-2-tod-1')` fails.

### Build the package

```bash
cd <repository root>
dpkg-buildpackage -b -us -uc
```

* `-b` — binary-only (no source package)
* `-us` / `-uc` — skip GPG signing of the source and changes files

Artifacts are written to the **parent** directory:

```
../libfprint-2-tod-vfs0090_0.96.91~f1_amd64.deb
../libfprint-2-tod-vfs0090-dbgsym_0.96.91~f1_amd64.ddeb
../libfprint-2-tod-vfs0090_0.96.91~f1_amd64.buildinfo
../libfprint-2-tod-vfs0090_0.96.91~f1_amd64.changes
```

### Verify the package before installing

```bash
dpkg-deb -c ../libfprint-2-tod-vfs0090_0.96.91~f1_amd64.deb
dpkg-deb -I ../libfprint-2-tod-vfs0090_0.96.91~f1_amd64.deb
```

Expected contents — a plugin, a udev rule, and docs, and nothing else:

```
./usr/lib/x86_64-linux-gnu/libfprint-2/tod-1/libfprint-tod-vfs009x.so
./usr/share/libfprint-tod-vfs0090/60-libfprint-2-tod-vfs0090.rules
./usr/share/doc/libfprint-2-tod-vfs0090/changelog.gz
./usr/share/doc/libfprint-2-tod-vfs0090/copyright
```

---

## 5. Installing and confirming

```bash
sudo apt install ./libfprint-2-tod-vfs0090_0.96.91~f1_amd64.deb
sudo systemctl restart fprintd
fprintd-list "$USER"
```

If `fprintd-list` shows the device, the plugin is being loaded correctly.

---

## 6. Why the `.ddeb` is not published

`dpkg-buildpackage` also emitted
`libfprint-2-tod-vfs0090-dbgsym_0.96.91~f1_amd64.ddeb` (~72 KB) — a detached
debug-symbols package.

It is **deliberately excluded** from the release because it has no practical
value for the target audience:

* Debug symbols are only useful alongside a configured `ddebs`/debug repository
  and a debugger workflow; there is no such repository behind this snapshot.
* It does not help anyone install or use the driver.
* It would add a second artifact users might install by mistake.

The `.buildinfo` and `.changes` files are likewise excluded — they are
build-metadata for archive uploads, not for end users.

Only the installable `.deb` is published as a Release asset.

### 6.1 GitHub release asset name: `~` becomes `.`

GitHub sanitizes the `~` character in release asset names, replacing it with
`.`. The asset therefore appears in the release as:

```
libfprint-2-tod-vfs0090_0.96.91.f1_amd64.deb
```

rather than the built name `libfprint-2-tod-vfs0090_0.96.91~f1_amd64.deb`.

This is **cosmetic and harmless**:

* The file **contents are byte-identical** — verified by downloading the
  published asset and comparing SHA-256 against the locally built package.
* The Debian version **inside** the package control metadata remains
  `0.96.91~f1`; only the filename on the release page differs.
* `apt`/`dpkg` do not care about the filename of a locally installed `.deb`.

`SHA256SUMS` in the release lists the asset under its **published** name so
that `sha256sum -c SHA256SUMS` works directly after downloading.

---

## 7. Files intentionally never committed or uploaded

The build environment also contained files that must **never** end up in this
repository. `.gitignore` encodes these exclusions:

| Category | Examples | Why |
| --- | --- | --- |
| Lenovo firmware blobs | `*.xpfwext`, `n1cgn08w.exe` | Proprietary, not redistributable |
| Sensor calibration / device state | calibration databases, Host GUID | Device-specific, sensitive |
| Fingerprint templates | prints, templates, captured images | Biometric data |
| Build output | `obj-*`, `debian/.debhelper/`, `*.substvars` | Reproducible from source |
| Release artifacts | `*.deb`, `*.ddeb`, `*.changes`, `*.buildinfo` | Published as Release assets instead |
| Machine config | `/etc/pam.d/*` | Machine-specific |
| Credentials | tokens, keys, `.netrc`, `.git-credentials` | Secrets |

---

## 8. Reproducibility notes

* The driver source is byte-identical to upstream `master` at the base commit;
  only `meson.build` differs, by one line.
* The `.deb` version string `0.96.91~f1` comes from the upstream
  `debian/changelog`.
* Package metadata (`Maintainer`, `Homepage`) still points at the upstream
  author and project, as it should — this snapshot does not claim authorship.
* To rebuild the exact release asset: apply the one-line `libudev` change,
  then run `dpkg-buildpackage -b -us -uc`.
