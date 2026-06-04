# Ubuntu GNOME Bluetooth Audio Profile Fix

A guide to resolving issues where Ubuntu treats a Bluetooth speaker as a low-quality handheld communication device (HSP/HFP) instead of a high-quality stereo speaker (A2DP), and forcing the "Configuration" dropdown menu to appear in GNOME Settings.

Tested on: **Ubuntu 22.04–26.04**, audio stack **PipeWire** (default since Ubuntu 22.04).

## Why This Happens

When a Bluetooth speaker also advertises a microphone profile (HSP/HFP), the system
auto-selects it over A2DP because HFP takes priority for "communication devices".
This locks the speaker into low-quality mono audio. The fix forces A2DP and tells
the Bluetooth service to hold multiple profiles simultaneously.

## Symptoms
* GNOME Settings > Sound > Output lacks a "Configuration" dropdown menu.
* The Bluetooth speaker is locked into a low-quality handheld communication mode.
* Selecting high-fidelity audio profiles fails or does not output sound due to routing conflicts between PulseAudio/PipeWire and Blueman.

## Step 1: Install Audio and Bluetooth Control Utilities
Open a terminal (`Ctrl + Alt + T`) and install the advanced volume control and Bluetooth management packages:
```bash
sudo apt update
sudo apt install pavucontrol blueman -y
```

## Step 2: Force High Fidelity Profile via Blueman
1. Open the **Blueman** application from your app drawer.
2. Right-click on your connected Bluetooth speaker.
3. Hover over **Audio Profile** and explicitly select **High Fidelity Playback**.

## Step 3: Edit the Bluetooth Configuration File
Modify the main Bluetooth service configuration to support multiple profiles simultaneously, preventing the system from automatically choosing the low-quality microphone channel.

**Option A — one-liner (recommended):**
```bash
sudo sed -i 's/#*MultiProfile.*/MultiProfile = multiple/' /etc/bluetooth/main.conf
```

**Option B — manual edit:**
```bash
sudo nano /etc/bluetooth/main.conf
```
1. Press `Ctrl + W` and search for the phrase: `MultiProfile`
2. Change the line (or uncomment it) to look exactly like this:
   ```text
   MultiProfile = multiple
   ```
3. Save the file by pressing `Ctrl + O`, then `Enter`.
4. Exit the editor by pressing `Ctrl + X`.

## Step 4: Restart Bluetooth and Audio Services
Restart the Bluetooth service and the PipeWire audio session to clear any stale
profile state:
```bash
sudo systemctl restart bluetooth
systemctl --user restart pipewire pipewire-pulse
```

> **PulseAudio (Ubuntu < 22.04):** replace the second command with
> `pulseaudio --kill && pulseaudio --start`

## Result

Open **GNOME Settings > Sound > Output**. The **"Configuration"** dropdown will now
be visible. Select **High Fidelity Playback (A2DP Sink)** to confirm full-quality
stereo audio is active.

If the dropdown still does not appear, disconnect and reconnect the speaker, then
repeat Step 2 in Blueman.
