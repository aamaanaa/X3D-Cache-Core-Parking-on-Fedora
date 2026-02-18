# X3D Core Parking / Driver in Linux

> ⚠️ Written and tested only on Fedora 43

---

<img width="599" height="428" alt="Screenshot From 2026-02-17 22-20-26" src="https://github.com/user-attachments/assets/05c1a20e-361a-4352-95fd-a5bc38f430dd" style="text-align:center;" />

> Don’t waste gaming performance like I did — this setup should be semi-default. Most Linux users are unaware that it isn’t enabled by default.

---

## TODO

* [ ] Instructions for compiling GameMode from Git (current release lacks X3D features)
* [ ] Refactoring and cleanup
* [ ] Add more screenshots (including BIOS options)
* [ ] Add GameMode test commands to verify functionality

---

## Doesn’t Linux already do this out of the box?

Nope. By default:

* The scheduler uses both CCDs (X3D and non-X3D), which can increase latency and reduce performance.
* On Fedora, the AMD X3D mode driver may not even be enabled, so the CPU does **not** automatically switch between cache and frequency modes.
* GameMode is also not configured to enable “cache” mode automatically.

Result: some games show better 1% lows and much smoother gameplay when the non-X3D CCD isn’t boosting or waking unnecessarily.

---

## For Fedora Users

Installing the latest Mesa from Git may boost FPS due to new features, as Mesa 26 has not landed yet on major distrubutions:
[https://gist.github.com/craimasjien/4519283aa2c170b93aff00b9f75aa7bf](https://gist.github.com/craimasjien/4519283aa2c170b93aff00b9f75aa7bf)
(*Thanks to craimasjien for the build script*)

---

## Required BIOS Settings

### Enable CPPC

Enable **CPPC** in BIOS under the “Driver” option. This allows sysfs to override the CPU mode.

### Disable C-States

Disabling C-states keeps non-X3D cores active instead of entering deep sleep.
This reduces wake-up latency when the scheduler uses those cores, helping minimize frame stutters and improve gaming latency on the X3D CCD.

---

## Enabling the `amd_x3d_vcache` Driver

Check the current mode:

```bash
$ cat /sys/bus/platform/drivers/amd_x3d_vcache/AMDI0101:00/amd_x3d_mode
frequency  # or cache
```

Check if the kernel has the driver installed:

```bash
$ grep AMD_3D_VCACHE /boot/config-$(uname -r)
CONFIG_AMD_3D_VCACHE=m
```

* **m** = installed as a kernel module (not loaded)

Load the module if needed:

```bash
$ sudo modprobe amd_3d_vcache
```

Manually prefer the X3D cache:

```bash
$ echo cache | sudo tee /sys/bus/platform/drivers/amd_x3d_vcache/AMDI0101:00/amd_x3d_mode
```

---

## Automating with Feral GameMode

Modify the GameMode config (may require compiling from Git):

```ini
[cpu]
; Pinning/parking can be enabled with "yes", "true", or "1". Disabled with "no", "false", or "0".
; Core ranges can be specified with comma-separated lists or ranges. E.g. "park_cores=1,8-15"
; Default: disable parking, enable pinning.

; AMD 9 9950X3D cache cores
pin_cores=0-7,16-23

; X3D cache mode configuration
amd_x3d_mode_desired=cache      ; GameMode sets this when gaming
amd_x3d_mode_default=frequency  ; Default system mode
```

**Notes:**

* Only works on CPUs with the AMD X3D driver (automatically detected).
* GameMode will switch the CPU to `cache` mode while gaming and restore `frequency` mode afterward.
* Threads 0–7 and 16–23 correspond to the X3D CCD on a Ryzen 9 9950X3D.

---

## Example: AMD Ryzen 9 9950X3D CCD Layout

`lstopo --no-io` output:

<img width="3218" height="1899" alt="image" src="https://github.com/user-attachments/assets/82520c7d-74be-4fc4-bf54-24ed9c464723" />

**Key points:**

* 2 chiplets with different specifications.
* Games should run on the CCD with extra cache (threads 0–7 and 16–23).
* Running across both CCDs introduces latency due to cross-CCD communication.

**Pinning recommendation:**

* **Threads 0–7 and 16–23** should be pinned to the X3D CCD.

Verify X3D cores have higher ranking:

```bash
$ grep -v /sys/devices/system/cpu/cpu*/cpufreq/amd_pstate_prefcore_ranking
```

Use `btop` during a game to confirm that the X3D cores are handling the load.

---

# Quick Visual Summary: CCDs & C-States for AMD Ryzen 9 9950X3D

Ryzen 9 9950X3D Overview
─────────────────────────
X3D CCD (0-7,16-23)       Non-X3D CCD
  ┌───────────────┐        ┌───────────────┐
  │ Game Workload │        │ System Tasks │
  │ 3D V-Cache    │        │ Smaller L3    │
  │ Preferred     │        │ Higher Clocks │
  └───────────────┘        └───────────────┘

C-States Enabled (Default)
─────────────────────────
Non-X3D cores → may enter deep sleep (C6)
Wake-up latency → small scheduling delays / frame stutters

C-States Disabled (Recommended for Gaming)
─────────────────────────
Non-X3D cores → stay active
Immediate execution for system tasks
Lower latency, smoother gaming
Higher idle power consumption
