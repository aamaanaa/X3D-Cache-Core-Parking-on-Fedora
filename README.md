# X3D core parking in linux

> [!WARNING]
> Written and tested only on Fedora 43

---

<img width="599" height="428" alt="Screenshot From 2026-02-17 22-20-26" src="https://github.com/user-attachments/assets/05c1a20e-361a-4352-95fd-a5bc38f430dd" style="text-align:center;" />


*in the above picture, we run a steam game, and only the X3D cores are active for best performance and latency*

---

## TODO
- [ ] instructions of how to compile game mode from git as it currenly lacks the x3d feature
- [ ] refactoring
- [ ] other stuff
- [ ] table of what cores to pin for what x3d cpu
- [ ] add more pictures (also from the bios option)
- [ ] add game mode test commands to see if it is working properly


### Enable CPPC in bios

**YOU NEED to enable something in your bios first:**
in the BIOS under the CPPC Option to the “Driver” Option. This will allow to override with the sysfs the used mode.


### Enabling the driver
See this what the current vallue is

```bash
$ cat /sys/bus/platform/drivers/amd_x3d_vcache/AMDI0101:00/amd_x3d_mode
frequency # or cache!
```

it may not have been installed in the kernel or even be loaded if it is installed...

```bash
$ grep AMD_3D_VCACHE /boot/config-$(uname -r)
CONFIG_AMD_3D_VCACHE=m
```

**m** = installed, but as a kernel module and thus not loaded

So, we need to load the kernel module:

```bash
$ sudo modprobe amd_3d_vcache
```

To manually prefer the X3D cache:

```bash
$ echo cache | sudo tee /sys/bus/platform/drivers/amd_x3d_vcache/AMDI0101:00/amd_x3d_mode
```

## automating with feral gamemode 

alter the config 

> **NOTE** you may have to compile gamemode from git!

```
[cpu]
; Parking or Pinning can be enabled with either "yes", "true" or "1" and disabled with "no", "false" or "0".
; Either can also be set to a specific list of cores to park or pin, comma separated list where "-" denotes
; a range. E.g "park_cores=1,8-15" would park cores 1 and 8 to 15.
; The default is uncommented is to disable parking but enable pinning. If either is enabled the code will
; currently only properly autodetect Ryzen 7900x3d, 7950x3d and Intel CPU:s with E- and P-cores.
; For Core Parking, user must be added to the gamemode group (not required for Core Pinning):
; sudo usermod -aG gamemode $(whoami)
; AMD 9 9950X3D cache is on the cores 0 - 7
; park_cores=
pin_cores=0-7,16-23

; AMD 3D V-Cache Performance Optimizer Driver settings
; These options control the cache mode for dual CCD X3D CPUs (7950x3d, 9950x3d, etc.)
; "frequency" mode prioritizes higher boost clocks, "cache" mode prioritizes 3D V-Cache performance
; Allows for dynamically shifting other processes onto a different CCD. E.g. amd_x3d_mode_default=cache may be
; preferred for some normal, non-game workloads that are better optimized for cache, but
; amd_x3d_mode_desired=frequency can shift everything but the game process to frequency CCD while GameMode is
; running, in conjunction with core pinning.
; Only works on systems with the AMD X3D mode driver (automatically detected)
; The desired mode is set when entering gamemode, default mode is restored when leaving
; frequency = default non gaming mode
amd_x3d_mode_desired=cache
amd_x3d_mode_default=frequency
```

**FOR AMD 9 9950X3D

output of `lstopo --no-io`

<img width="3218" height="1899" alt="image" src="https://github.com/user-attachments/assets/82520c7d-74be-4fc4-bf54-24ed9c464723" />

the 9950x3d has 2 chiplests with diffrent specs, it is widely agreed upon that games should be ran on the ccd with extra cache which are threads 0-7 and 16-23. having a game run across both chipsets makes games lose prefroamnce due to a latency increase from the 2 ccds having to talk to eacher,


**0-7,16-23** threads must be pinned to threads 0-7 and 16-23 on a AMD 9 9950x3D cpu.



You may also use this command to check if it is working, the x3d cores should have a higher ranking:

```bash
$ grep -v /sys/devices/system/cpu/cpu*/cpufreq/amd_pstate_prefcore_ranking
```

if it all went well, during a game and using the `btop` command, you should see the result in the picture above in this repo.
