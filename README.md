# libfprint TOD driver for Validity VFS0090 — ThinkPad T460s on Kubuntu / Ubuntu 26.04

`libfprint` TOD (Touch OEM Driver) plugin for the **Validity Sensors VFS0090**
fingerprint reader (**USB ID `138a:0090`**), as found in the **Lenovo ThinkPad T460s**
(and other 2016-era ThinkPads).

This repository is a **repackaged downstream snapshot** of the upstream driver
[`3v1n0/libfprint-tod-vfs0090`](https://github.com/3v1n0/libfprint-tod-vfs0090)
by **Marco Trevisan (Treviño)**, with one build-system compatibility fix
required to compile on **Kubuntu 26.04 / Ubuntu 26.04**, plus build and
troubleshooting documentation.

> **Upstream project:** <https://github.com/3v1n0/libfprint-tod-vfs0090>
> (mirror of <https://gitlab.freedesktop.org/3v1n0/libfprint-tod-vfs0090>)
> All driver code and copyright belong to the upstream author.
> See [Licensing](#licensing) below.
> The original, unmodified upstream README is preserved as
> [`README.upstream.md`](README.upstream.md).

---

## Table of contents

1. [Hardware: ThinkPad T460s / Validity VFS0090 (`138a:0090`)](#1-hardware-thinkpad-t460s--validity-vfs0090-138a0090)
2. [Why the stock Ubuntu/Kubuntu libfprint can't see it](#2-why-the-stock-ubuntukubuntu-libfprint-cant-see-it)
3. [Why `validity-sensors-tools` initialization is mandatory](#3-why-validity-sensors-tools-initialization-is-mandatory)
4. [Installing and initializing the sensor](#4-installing-and-initializing-the-sensor)
5. [Installing this TOD driver](#5-installing-this-tod-driver)
6. [Building from source on Kubuntu 26.04](#6-building-from-source-on-kubuntu-2604)
7. [Enrolling a finger with `fprintd-enroll`](#7-enrolling-a-finger-with-fprintd-enroll)
8. [Verifying with `fprintd-verify`](#8-verifying-with-fprintd-verify)
9. [Enabling fingerprint auth for sudo / KDE via `pam-auth-update`](#9-enabling-fingerprint-auth-for-sudo--kde-via-pam-auth-update)
10. [Known limitations and troubleshooting](#10-known-limitations-and-troubleshooting)

---

## 1. Hardware: ThinkPad T460s / Validity VFS0090 (`138a:0090`)

The ThinkPad T460s ships a Validity Sensors fingerprint reader on the USB bus:

```console
$ lsusb | grep -i 138a
Bus 001 Device 004: ID 138a:0090 Validity Sensors, Inc. VFS7500 Touch Fingerprint Sensor
```

Note that `lsusb` reports this chip as a **VFS7500**; the `138a:0090` USB ID is
what matters, and the corresponding libfprint TOD driver is the **`vfs0090`**
driver in this repository (module name `libfprint-tod-vfs009x`).

The sibling device `138a:0097` is also supported by this same driver and is
handled by the same source file. `138a:0097` additionally supports
**match-on-chip**, which `138a:0090` does not — see
[Known limitations](#10-known-limitations-and-troubleshooting).

---

## 2. Why the stock Ubuntu/Kubuntu libfprint can't see it

The VFS0090 is **not a USB-class-compliant fingerprint reader**. It presents no
usable standard interface for image capture: the host must speak a proprietary
protocol and, critically, the sensor must first be loaded with a proprietary
firmware blob and cryptographically **initialized/pair**ed with the host.

Because of this, the device is a *TOD* ("Touch OEM Driver") device. Upstream
`libfprint` deliberately supports such OEM devices only through **out-of-tree
TOD plugins**, which are loaded from a driver directory at runtime rather than
compiled into the core library. The consequences:

* Stock `libfprint` / `fprintd` on Ubuntu and Kubuntu ships **no** `vfs0090`
  TOD plugin, so the reader is simply invisible to `fprintd`.
* `fprintd-enroll` fails with a message along the lines of
  `No devices available` / `Impossible to enroll: GDBus.Error:net.reactivated.Fprint.Error.NoSuchDevice`.
* This is expected behaviour, **not** a hardware fault and **not** something a
  kernel update fixes.

Three independent pieces are therefore required:

1. A libfprint build with **TOD support** (`libfprint-2-tod1`),
2. this **TOD driver plugin** installed into the TOD drivers directory, and
3. the sensor itself **initialized** (section 3) — the plugin is useless on an
   uninitialized device.

---

## 3. Why `validity-sensors-tools` initialization is mandatory

> **The driver in this repository only works on a sensor that has already been
> initialized with `validity-sensors-tools`.**

The VFS0090 boots into a state where it does not yet accept normal capture
operations. Before the reader can be used by any Linux driver, it must be:

1. **Firmware-loaded** — the proprietary firmware blob (downloaded from Lenovo)
   is pushed to the device, and
2. **Initialized / paired** — a per-device initialization and pairing step that
   establishes the host↔sensor relationship and stores device state.

`validity-sensors-tools` performs both steps. Until that has been done, this
driver will load but the device will not enumerate as usable and enroll will
fail.

This initialization is **persistent device state**, not something you redo on
every boot — but it is tied to the specific sensor and its stored state. If the
device is reset or that state is lost, you must re-initialize.

The upstream project documents the same requirement:

> It only works if the device has been initialized using
> [validity-sensors-tools](https://snapcraft.io/validity-sensors-tools)

---

## 4. Installing and initializing the sensor

The supported path is the `validity-sensors-tools` **snap**, which bundles the
firmware-handling and initialization tooling:

```bash
sudo snap install validity-sensors-tools

# Give the snap raw USB access — without this it cannot see the sensor
sudo snap connect validity-sensors-tools:raw-usb

# Initialize (firmware load + pairing) the sensor
sudo validity-sensors-tools.initializer

# Optional: confirm the sensor responds by blinking its LED
sudo validity-sensors-tools.led-test

# For 138a:0097 only — enroll into the chip to enable match-on-chip.
# This is NOT supported on 138a:0090 (the T460s), skip it there.
# sudo validity-sensors-tools.enroll --finger-id 0

# List the other available subcommands
validity-sensors-tools --help
```

If you prefer not to use snap, the same tooling can be run from source via
[`3v1n0/python-validity`](https://github.com/3v1n0/python-validity).

> **Do not commit or share** the firmware blobs (`*.xpfwext`, Lenovo's
> `n1cgn08w.exe` installers) or the device state / calibration / Host GUID files
> produced by this tooling. They are proprietary and/or device-specific. See
> [`.gitignore`](.gitignore).

---

## 5. Installing this TOD driver

### Recommended: install the Debian package

A prebuilt package for Ubuntu 26.04 / amd64 is published on the
[**Releases**](../../releases) page:

```bash
sudo apt install ./libfprint-2-tod-vfs0090_0.96.91~f1_amd64.deb
```

`apt` resolves the dependencies (`libfprint-2-tod1`, `libnss3`, `libssl3t64`)
and places the plugin in the correct TOD drivers directory automatically. The
package also installs the udev rule granting the `plugdev` group access to the
device.

Verify the plugin landed and that the udev rule is present:

```bash
ls -l /usr/lib/x86_64-linux-gnu/libfprint-2/tod-1/libfprint-tod-vfs009x.so
ls -l /usr/share/libfprint-tod-vfs0090/60-libfprint-2-tod-vfs0090.rules
```

Then restart the daemon and confirm the device is now visible:

```bash
sudo systemctl restart fprintd
fprintd-list "$USER"
```

### ⚠️ Do NOT hand-copy the `.so` into `/usr/lib`

Do not manually `cp` `libfprint-tod-vfs009x.so` into `/usr/lib/...`. Doing so:

* bypasses dependency resolution and versioned package management,
* leaves the udev rule uninstalled (so the device stays permission-denied),
* cannot be cleanly upgraded or removed, and
* makes the failure modes very hard to debug.

Always use the `.deb` (or build and install via `meson install`, section 6),
so that dpkg/apt tracks the files.

---

## 6. Building from source on Kubuntu 26.04

### Why we build from source here

Upstream ships an Ubuntu PPA (`ppa:3v1n0/libfprint-vfs0090`). **That PPA has no
`resolute` series**, so it cannot be used on Ubuntu/Kubuntu 26.04 — there is no
26.04 build of the package to install. Building from source is therefore the
supported route on 26.04, and the prebuilt `.deb` in Releases was produced
exactly this way.

> Ubuntu 26.04 is codenamed **resolute**; the PPA publishes only older series.

### The `udev` → `libudev` compatibility fix

Upstream `meson.build` declares the udev dependency as:

```meson
udev_dep = dependency('udev')
```

On Ubuntu 26.04 there is **no pkg-config file named `udev.pc`**. The udev
development metadata is provided by `libudev-dev`, whose pkg-config file is
named `libudev.pc`. Meson's `dependency('udev')` therefore fails:

```
Run-time dependency udev found: NO (tried pkgconfig and cmake)
meson.build:15:0: ERROR: Dependency "udev" not found, tried pkgconfig and cmake
```

The fix is a one-line change in `meson.build`:

```diff
-udev_dep = dependency('udev')
+udev_dep = dependency('libudev')
```

`libudev` is the correct pkg-config name for the same library on modern
Debian/Ubuntu; it still exposes the `udevdir` variable that the rule
installation step needs, so no other change is required. This single change is
the only source modification in this snapshot — see
[`BUILD-UBUNTU-26.04.md`](BUILD-UBUNTU-26.04.md) for the full write-up of the
build and troubleshooting session.

### Build steps

```bash
sudo apt update
sudo apt install -y \
    build-essential debhelper devscripts meson pkg-config \
    libfprint-2-tod-dev libfprint-2-dev libglib2.0-dev libgusb-dev \
    libpixman-1-dev libssl-dev libnss3-dev libusb-1.0-0-dev \
    libudev-dev udev

# From the repository root
dpkg-buildpackage -b -us -uc
```

The resulting `libfprint-2-tod-vfs0090_*.deb` is written to the parent
directory. Install it as in section 5.

Alternatively, for a direct non-packaged build:

```bash
meson setup _build
ninja -C _build
sudo ninja -C _build install
```

---

## 7. Enrolling a finger with `fprintd-enroll`

Confirm the device is visible first:

```bash
fprintd-list "$USER"      # should list the VFS0090 device
```

Enroll a single finger:

```bash
fprintd-enroll -f right-index-finger "$USER"
```

Swipe the finger repeatedly when prompted until enrollment completes. To enroll
all ten fingers in one pass:

```bash
for finger in {left,right}-{thumb,{index,middle,ring,little}-finger}; do
    fprintd-enroll -f "$finger" "$USER"
done
```

On KDE Plasma you can alternatively use
**System Settings → Users → (your user) → Fingerprint Login**.

Check what is currently enrolled:

```bash
fprintd-list "$USER"
```

---

## 8. Verifying with `fprintd-verify`

```bash
fprintd-verify "$USER"
```

Or verify a specific enrolled finger:

```bash
fprintd-verify -f right-index-finger "$USER"
```

A successful match prints `verify-result: verify-match`; a non-match prints
`verify-no-match`. `verify-no-match` is usually a placement/quality issue —
re-position your finger and retry, and see the troubleshooting section.

Useful while tuning: watch the daemon log for comparison scores.

```bash
journalctl -u fprintd -f
```

---

## 9. Enabling fingerprint auth for sudo / KDE via `pam-auth-update`

Enrolling a finger only *stores* the print — it does not by itself let you use
it to authenticate. PAM must be told to consult `fprintd`.

`libpam-fprintd` (installed with fprintd on Debian/Ubuntu) ships a PAM profile,
so the supported way to enable it is `pam-auth-update`:

```bash
sudo apt install -y libpam-fprintd
sudo pam-auth-update
```

In the curses dialog, enable **Fingerprint authentication**, then confirm. This
rewrites the managed PAM stacks (`/etc/pam.d/common-auth` and friends) through
the proper Debian profile mechanism.

Verify:

```bash
grep -n fprint /etc/pam.d/common-auth
```

Test in a **new** terminal (keep your current session open as a fallback):

```bash
sudo -k && sudo -v      # prompts for fingerprint, password remains as fallback
```

### KDE / SDDM / lock screen

* **KDE polkit prompts** and **KDE screen unlocking** pick up the PAM change
  automatically once fingerprint auth is enabled, provided
  `libpam-fprintd` is installed.
* **SDDM login** is a separate PAM stack. To enable fingerprint at the login
  screen, also configure the `sddm` PAM service (via `pam-auth-update` it is
  generally included; if not, edit `/etc/pam.d/sddm` and add
  `auth sufficient pam_fprintd.so` **above** the password line).
* Always keep password authentication enabled as a fallback — do not make
  fingerprint the only method.

> **Do not copy whole `/etc/pam.d` files into this repository.** PAM
> configuration is machine-specific. Only the commands above belong in
> documentation.

---

## 10. Known limitations and troubleshooting

### Known limitations

* **`138a:0090` has no match-on-chip.** Unlike `138a:0097`, the T460s sensor
  cannot store fingerprints in the chip. Matching is done in software by
  libfprint using its image-comparison algorithm, which is slower and more
  sensitive to finger placement and image quality.
* **Requires prior initialization.** Without `validity-sensors-tools`
  initialization (section 3) the driver cannot work at all.
* **Enrollment quality matters.** Enroll the same finger several times and use
  a clean, dry finger. Poor enrollment is the most common cause of
  `verify-no-match`.
* **`bz3_threshold` tuning.** Matching sensitivity is governed by
  `bz3_threshold` (currently `12`). Lower values match more readily but are less
  secure; higher values are stricter. Upstream explicitly invites tuning
  reports for `138a:0090`.
* **Not a security boundary for high-value auth.** Treat convenience
  fingerprint login as such and keep strong passwords as fallback.

### Troubleshooting

**Device not listed at all**

```bash
lsusb | grep 138a                  # is the reader on the bus?
fprintd-list "$USER"               # does fprintd see a device?
systemctl status fprintd
journalctl -u fprintd -b --no-pager | tail -50
```

* If `lsusb` shows nothing, the issue is hardware/BIOS, not this driver.
* If `lsusb` shows `138a:0090` but `fprintd` does not, check the plugin is
  installed and in the right directory:

  ```bash
  pkg-config --variable=tod_driversdir libfprint-2-tod-1
  ls -l "$(pkg-config --variable=tod_driversdir libfprint-2-tod-1)"
  ```

* Confirm the TOD-enabled libfprint is present:

  ```bash
  dpkg -l | grep -E 'libfprint-2-tod1|libfprint-2-2'
  ```

**Permission denied on the device**

```bash
ls -l /dev/bus/usb/*/*      # look at the 138a device node
```

It should be group `plugdev`, mode `0660`. If not, the udev rule is missing or
not reloaded:

```bash
sudo cp 60-libfprint-2-tod-vfs0090.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Also confirm your user is in `plugdev`:

```bash
groups "$USER" | grep plugdev
sudo usermod -aG plugdev "$USER"     # then log out and back in
```

**Build fails with `Dependency "udev" not found`**

This is the 26.04 compatibility issue — apply the
`dependency('udev')` → `dependency('libudev')` fix from section 6 and install
`libudev-dev`.

**`verify-no-match` on a good finger**

* Re-enroll, capturing the finger from slightly different angles each swipe.
* Ensure the sensor surface is clean and your finger is dry.
* Watch scores while verifying:

  ```bash
  journalctl -u fprintd -f
  ```

  Look for `fpi_img_compare_print_data] score N` lines. If scores hover just
  below the threshold, enrolling more samples usually helps.

**Enrollment hangs or the LED never lights**

The sensor is almost certainly not initialized. Re-run:

```bash
sudo validity-sensors-tools.initializer
sudo validity-sensors-tools.led-test
```

**`fprintd` keeps prompting for password instead of fingerprint**

Confirm the PAM profile is active:

```bash
grep -n fprint /etc/pam.d/common-auth
```

If absent, re-run `sudo pam-auth-update` and enable *Fingerprint
authentication*.

---

## Repository contents

| Path | Purpose |
| --- | --- |
| `vfs0090.c`, `vfs0090.h` | Driver source (upstream, unmodified) |
| `meson.build` | Build definition — **contains the one `libudev` fix** |
| `60-libfprint-2-tod-vfs0090.rules` | udev rule for `138a:0090` / `138a:0097` |
| `debian/` | Debian packaging (upstream) |
| `.gitignore` | Excludes build output, firmware, device state, biometric data |
| `README.upstream.md` | Original upstream README, preserved verbatim |
| `BUILD-UBUNTU-26.04.md` | Build/troubleshooting log for 26.04 |
| `README.md` | This document |

Built `.deb` files are **not** committed — they are published as
[Release](../../releases) assets.

---

## Licensing

This repository contains **no new license**. Upstream licensing and copyright
are preserved as-is:

* `debian/copyright` declares **LGPL-3.0+**, copyright
  **2020 Marco Trevisan (Treviño) <marco@ubuntu.com>**, upstream
  <https://gitlab.freedesktop.org/3v1n0/libfprint-tod-vfs0090>.
* `meson.build` declares `license: 'LGPLv2.1+'`.

These two upstream statements disagree; this snapshot does not attempt to
resolve the discrepancy and adds no license of its own. For the authoritative
terms, consult the upstream project. The full LGPL text is on Debian systems at
`/usr/share/common-licenses/LGPL-3`.

---

## Credits

All driver work is by **[Marco Trevisan (Treviño)](https://github.com/3v1n0)**,
building on the reverse-engineering of
[nmikhailov/Validity90](https://github.com/nmikhailov/Validity90) and
[uunicorn](https://github.com/uunicorn/)'s
[python-validity](https://github.com/uunicorn/python-validity) and
[synaWudfBioUsb-sandbox](https://github.com/uunicorn/synaWudfBioUsb-sandbox).

This snapshot adds only the Ubuntu 26.04 build compatibility fix and
documentation.
