# ASUS Motherboard MT7927 Wi-Fi 7 + Bluetooth 5.4 Fix on Linux

A guide to activating the MediaTek MT7927 (Filogic 380) Wi-Fi 7 and Bluetooth 5.4
hardware found on ASUS ROG/ProArt X870E and similar boards when Linux shows the
device as unclaimed after installation.

Tested on: **Ubuntu 26.04**, kernel **7.0.0-22-generic**, ASUS ROG Strix X870E-E.
The same steps apply to any distribution with kernel 6.17+.

---

## Why This Happens

The Linux kernel (7.0+) recognises the MT7927 PCIe device but cannot initialise it
because two things are missing:

1. **Kernel module patches** — the `mt7925e` driver does not yet carry MT7927 device
   IDs or hardware init code in mainline. The Bluetooth (`btusb`) driver likewise lacks
   the MT6639 USB device IDs used by the BT half of the chip.
2. **Firmware microcode** — the `linux-firmware` package does not yet ship the three
   `.bin` blobs the driver needs. The upstream merge request is open but pending review.

Both gaps are filled by the community DKMS package `mediatek-mt7927-dkms`.

---

## Affected Hardware

Any board with PCI ID `14c3:7927` (Wi-Fi) and USB ID `0489:e13a` or `13d3:3588`
(Bluetooth). Examples:

- ASUS ROG Crosshair X870E Hero / Glacial
- ASUS ROG Strix X870E-E / X870-I
- ASUS ProArt X870E-Creator WiFi (rev 2)
- ASUS ROG STRIX B850-E Gaming WiFi
- Gigabyte X870E / Z790 Aorus series
- MSI MEG X870E ACE MAX
- Lenovo Legion Pro 7 (16ARX9, 16AFR10H)

Check yours:
```bash
lspci -nn | grep -i 14c3        # Wi-Fi (PCIe)
lsusb | grep -iE '0489|13d3'   # Bluetooth (USB)
```

---

## Step 1 — Confirm the Device is Unclaimed

```bash
sudo lshw -C network
```

Look for a wireless entry with `configuration: ...` absent or `*-network UNCLAIMED`.
That confirms the kernel sees the hardware but has no driver loaded for it.

```bash
lsusb | grep -iE '0489|13d3|0e8d'
```

If no output, Bluetooth is completely offline — expected until Wi-Fi is working.

---

## Step 2 — Install Build Dependencies

```bash
sudo apt update
sudo apt install dkms make gcc python3 linux-headers-$(uname -r)
```

---

## Step 3 — Clone the DKMS Package

```bash
git clone https://github.com/jetm/mediatek-mt7927-dkms.git
cd mediatek-mt7927-dkms
```

The repository contains kernel patches, build scripts, and a firmware extraction tool.
It does **not** contain any pre-built binaries.

---

## Step 4 — Download the Kernel Tarball and ASUS Firmware Driver

`make download` fetches two things:

- A Linux 7.0 kernel source tarball from kernel.org (~150 MB) — used only to extract
  the mt76 Wi-Fi and btusb Bluetooth source trees before patching.
- The official ASUS driver ZIP from the ASUS CDN (~23 MB) — contains `mtkwlan.dat`,
  a MediaTek firmware container holding the three `.bin` microcode blobs Linux needs.

```bash
make download
```

> **Note:** The Windows `.sys`/`.dll`/`.inf` files inside the ASUS ZIP are never used.
> Only the platform-neutral firmware blobs embedded in `mtkwlan.dat` are extracted.

---

## Step 5 — Prepare the Patched Source Tree

```bash
make sources
```

This step:
- Extracts three firmware blobs from `mtkwlan.dat` into `_build/firmware/`:
  - `BT_RAM_CODE_MT6639_2_1_hdr.bin`
  - `WIFI_MT6639_PATCH_MCU_2_1_hdr.bin`
  - `WIFI_RAM_CODE_MT6639_2_1.bin`
- Extracts the mt76 Wi-Fi and btusb Bluetooth source trees from the kernel tarball.
- Applies 19 Wi-Fi patches and 9 Bluetooth patches to the extracted sources.

---

## Step 6 — Install into DKMS

```bash
sudo make install
```

This copies the patched source tree into `/usr/src/mediatek-mt7927-2.12/` and places
the three firmware blobs into `/usr/lib/firmware/mediatek/mt7927/`.

---

## Step 7 — Register, Build, and Activate the Kernel Modules

First confirm the version installed by `make install`:

```bash
grep PACKAGE_VERSION /usr/src/mediatek-mt7927-*/dkms.conf
# e.g. PACKAGE_VERSION="2.12"
```

Then substitute that version throughout:

```bash
sudo dkms add mediatek-mt7927/2.12
sudo dkms build mediatek-mt7927/2.12
sudo dkms install mediatek-mt7927/2.12
```

The build step compiles the patched `mt7925e` (Wi-Fi) and `btusb`/`btmtk` (Bluetooth)
kernel modules against your running kernel headers. This takes 1–2 minutes.

### Secure Boot

If Secure Boot is enabled, `dkms install` will prompt for a **MOK enrollment password**.
Set any short password — it is used exactly once:

1. Set the password when prompted.
2. Let the install complete (`Running depmod... done.`).
3. Reboot.
4. At the blue **MOK Manager** screen (appears before Ubuntu loads):
   - Select **Enroll MOK** → **Continue**
   - Enter the password you set
   - Select **Yes** → **Reboot**

The signing key is now trusted by your UEFI firmware. The modules will load on every
subsequent boot without further prompts.

> The MOK password is unrelated to your login password, disk encryption, or any other
> credentials. It is a one-time UEFI key enrollment token.

---

## Step 8 — Load the Modules

After the MOK reboot (or immediately if Secure Boot is disabled):

```bash
sudo modprobe -r mt7925e mt7921e btusb
sudo modprobe mt7925e
sudo modprobe btusb
```

---

## Step 9 — Verify

```bash
lspci -nn | grep -i 14c3       # should show driver: mt7925e
lsusb | grep -iE '0489|13d3'  # Bluetooth USB device should now appear
ip link                        # wireless interface (e.g. wlp9s0) should be listed
```

Expected output when working:
```
# lspci
09:00.0 Network controller [0280]: ... [14c3:7927] ... Kernel driver in use: mt7925e

# lsusb
Bus 001 Device 004: ID 0489:e13a Foxconn / Hon Hai ...

# ip link
4: wlp9s0: <BROADCAST,MULTICAST> mtu 1500 ... state DOWN
```

Wi-Fi and Bluetooth should now appear in GNOME Settings. If Bluetooth shows as soft-blocked:

```bash
rfkill unblock bluetooth
```

If WPA authentication fails on 5/6 GHz on first attempt:

```bash
nmcli connection modify <ssid> connection.auth-retries 3
```

### Cleanup (optional)

Once the modules are confirmed working, the build artefacts are no longer needed:

```bash
cd mediatek-mt7927-dkms
make clean                  # removes _build/ (~1.5 GB of extracted kernel source)
rm linux-7.0.tar.xz         # removes the kernel tarball (~150 MB)
rm DRV_WiFi_MTK_MT7925_MT7927_*.zip  # removes the ASUS driver ZIP (~23 MB)
```

DKMS retains only the compiled modules and the source tree under
`/usr/src/mediatek-mt7927-<version>/`, which is needed for future kernel rebuilds.

---

## Known Issues

**Bluetooth USB device disappears after module reload or DKMS upgrade**
The MT6639 BT firmware can lock up during a module reload, causing the USB device to
vanish from `lsusb`. This persists across reboots and affects Linux and Windows.
Fix: shut down completely, unplug the PSU cable (or switch off at the rear), wait 10
seconds, then power back on. A regular reboot is not enough.

**Low upload throughput at 160 MHz with Wi-Fi 7 clients**
The EHT data path in the current firmware adds overhead that roughly halves upload at
160 MHz. Disable EHT to recover throughput by adding to your wpa_supplicant config:
```text
disable_eht=1
```

---

## Updating the Drivers

### After a Kernel Upgrade

DKMS rebuilds modules automatically on every kernel install. No manual action is needed.
To verify:

```bash
dkms status
```

All installed kernels should show `mediatek-mt7927/2.12: installed`.

If a module is missing for a new kernel, rebuild manually:

```bash
sudo dkms install mediatek-mt7927/2.12
```

### Upgrading to a New Package Version

When a new version of `mediatek-mt7927-dkms` is released:

```bash
cd mediatek-mt7927-dkms
git pull

# Remove the old version from DKMS
sudo dkms remove mediatek-mt7927/2.12 --all

# Clean the previous build
make clean

# Download new assets (kernel tarball and/or driver ZIP if changed)
make download

# Rebuild and reinstall
make sources
sudo make install
sudo dkms add mediatek-mt7927/<new-version>
sudo dkms build mediatek-mt7927/<new-version>
sudo dkms install mediatek-mt7927/<new-version>

# Reload modules
sudo modprobe -r mt7925e mt7921e btusb
sudo modprobe mt7925e
sudo modprobe btusb
```

Replace `<new-version>` with the value of `PACKAGE_VERSION` in `dkms.conf`.

### Checking for Upstream Mainline Inclusion

Once the patches are accepted into the Linux kernel and firmware repositories, this DKMS
package will no longer be needed. Track upstream status at:
- Wi-Fi + BT driver patches: https://github.com/jetm/mediatek-mt7927-dkms/issues/15
- BT firmware MR: https://gitlab.com/kernel-firmware/linux-firmware/-/merge_requests/946

When mainline support lands, remove the DKMS package:

```bash
sudo dkms remove mediatek-mt7927/<version> --all
sudo rm -rf /usr/src/mediatek-mt7927-<version>
```

The standard `linux-firmware` package will then provide the firmware blobs automatically.

---

## See Also

- [Ubuntu GNOME Bluetooth Audio Profile Fix](./fix_bt-audio-profile.md) — once Wi-Fi
  and Bluetooth are working, the system may default to low-quality HSP/HFP audio on
  Bluetooth speakers. That guide fixes the A2DP profile selection.
