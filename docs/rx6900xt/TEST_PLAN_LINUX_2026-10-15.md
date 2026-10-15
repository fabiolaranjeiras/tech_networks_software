# RX 6900 XT — Linux test plan for a remote executor agent (2026-10-15)

**Audience:** an AI agent with SSH access to a *controller* computer on the LAN, which in turn reaches the *target* PC after the owner boots it from the USB SSD into Linux.
**Author:** Claude Code (documentation steward for the RX 6900 XT records). **Status:** NOT yet executed. Nothing in this file is a result.
**Read first (in this order):**
1. `RTX_Ryzen9/AMD_Radeon_RX_6900_Linux_tests.md` — the single source of truth about the card (facts, evidence, open questions).
2. `RTX_Ryzen9/temp/gpu_test/*.log` — the raw logs from the first Linux session (2026-10-15).
3. `RTX_Ryzen9/temp/updater/RX6900XT_USB_TEST_AGENT_MISSION.md` — original safety brief (still binding; the invariants are repeated in section 2).
4. This file.

You decide *how* to implement. The goals, safety invariants, measurements and record format below are the contract. Where a flag or tool behavior is not verified here, run `--help` (or read the upstream docs) on the installed version and record the version. **Never invent flags.**

---

## 1. Verified facts you can rely on (and facts you must not rely on)

### 1.1 Hardware and software (from the logs and the owner)
| Item | Value | Source |
|---|---|---|
| Target hostname | `rx6900xt-test` (Debian 13, headless, booted from USB SSD) | `test_results.log` |
| GPU | XFX Speedster MERC 319 **Black**, Navi 21 XTX, PCI `1002:73bf` rev c0, subsystem `1eae:6901`, bus `09:00.0` | `test_results.log`, dmesg |
| VBIOS | `113-69XB6SSB1-D01` (ATOM BIOS), SMU fw 58.90.0, amdgpu 3.61.0, 80 CUs, VRAM 16368M GDDR6 256-bit | dmesg, `furmark.log` |
| **Resizable BAR** | **`BAR=256M`** in Linux (ReBAR not active); Windows HWiNFO (2026-02-03) said the same | dmesg |
| Power cap | `PPT cap = 289 W` (hwmon). Idle: 6–12 W, fan 0 RPM, edge 33–42 °C | `sensors` in `test_results.log` |
| Sensor limits | edge crit 100 / emerg 105 °C; junction crit 110 / emerg 115; mem crit 100 / emerg 105 | `sensors` |
| Kernel | `6.12.107+deb13` (first sessions), `6.12.111+deb13` (current). **It changed between sessions.** | logs |
| Vulkan | RADV NAVI21 (API 1.4.305) as device 1; **`llvmpipe` (CPU) is also listed** | `memtest_vulkan.log`, `furmark.log` |
| PSU / cabling | Corsair RM1000x (vendor spec recorded by another agent: single +12 V rail, 83.3 A). **The card is fed by two separate PCIe cables (owner confirmed 2026-10-15). The "Y/daisy-chain cable" hypothesis is therefore refuted.** | owner; `maquinas.json` |
| Motherboard BIOS | 3636 (2026-01-04), read live via DMI by another agent on 2026-10-15 (older records said 2423 and 3621) | dossier / mission edits |
| Host | Ryzen 9 5900X, ASUS TUF B550M-PLUS, 64 GB. Network: wired `enp6s0` (r8169, 1 Gbps) and Wi-Fi `wlp5s0`; **IP is assigned by DHCP and unknown in advance** | dmesg |

### 1.2 What the first session actually proved (read the logs, not the summaries)
| Test | What the raw log shows |
|---|---|
| `memtest_vulkan` v0.5.0 | Started 12:23:14Z. ~12.5 min (24 lines × 30 s + warm-up), **12,942 iterations, no error line, "Standard 5-minute test PASSed"**. 448 GB/s write, ~430–436 GB/s check. One 30-s window (iteration ~3214→3604) wrote only 263.6 GB/s (not explained). **The tool has no memory-size argument** (upstream: "memory size is not configurable"), so the earlier "12 GB" label is unsupported; per-iteration volume was 10.9 GB written / 14.5 GB checked. |
| `llama-bench` Qwen2.5-0.5B Q4_K_M `-p 512 -n 128 -r 3` | pp512 18,469.90 ± 2,587 t/s, tg128 552.4 ± 36 t/s. **Total GPU time ≈ 1–2 seconds.** Passed. |
| `llama-bench` Qwen2.5-3B Q4_K_M `-p 512,1024 -n 128 -r 2` | pp512 completed (4,266.88 ± 37.3 t/s). **The machine reset during pp1024, i.e. after only a few seconds of GPU compute. It was NOT a "sustained" load.** |
| FurMark 2.10.2 (`--demo furmark-vk --width 1920 --height 1080 --no-score-box --max-time 15`) | **Never rendered.** Log: `cannot open default display!`, shutdown after 1 s. The claimed "1,406 frames / 95 FPS" is **not supported by any log**. FurMark needs an X/Wayland display. |
| `vkmark` | No log exists. The claimed score (13,966) is unverified. |
| `run_benchmarks.sh` | Executed 3× (08:20, 09:15, 12:10). **Aborted at step 2 every time**: first `CMake Error ... memtest_vulkan ... does not contain CMakeLists.txt` (memtest_vulkan is a Rust/cargo project, not CMake), then `fatal: destination path 'memtest_vulkan' already exists` (`set -e`). Steps 3–5 never ran. The reported tests were typed by hand; commands for most are not recorded. |
| `dmesg_previous_boot.log` | A **~23-second boot (kernel .107)** that ends in `systemd-shutdown`. It contains **no GPU test and no GPU error**. It is *not* the crash session. Crash-time kernel logs are therefore **missing** (see test A2). |
| `dmesg_current_boot.log` | Kernel .111. No amdgpu/AER/MCE errors. `FAT-fs (sda5): Volume was not properly unmounted` (likely the ESP; an unclean shutdown happened at some point). Wall-clock timestamps in the logs are inconsistent with the `date` output of the scripts; use monotonic times. |
| Telemetry under load (272 W, 56/75 °C, fan 1174 RPM, "3.00 GHz") | **Appears only in chat narrative, in no log.** Treat as unverified. |

### 1.3 Working hypotheses (all unproven — the plan exists to separate them)
- **H1** Acute current/power transient on the GPU core rail (dense FP16/dot-product shader work such as GEMM) trips a protection (card VRM or PSU OCP/OPP) within seconds. Memory-bound work (`memtest_vulkan`) at similar board power does not.
- **H2** Defect in the card's power stage (component degraded).
- **H3** PSU / PSU output port / motherboard PCIe slot power path.
- **H4** PCIe signalling or Resizable BAR state.
- **H5** Software (Mesa/RADV compute bug) — unlikely to power-cycle a machine, but cheap to probe (different kernel/Mesa is *not* part of this plan; do not change versions mid-campaign).

---

## 2. Safety invariants (binding; copied from the original mission and extended)

1. **Never touch the Windows SSD (Lexar NM790 4 TB) or any disk other than the Linux root/ESP on the USB SSD.** Do not format, repartition, `fsck -y`, or write to anything else. Identify disks by serial/UUID, never by `/dev/sdX` alone.
2. **No firmware writes of any kind**: no VBIOS flash, no SSD/enclosure firmware, no BIOS update. Reading identity is fine. No BIOS setting changes — list them for the owner instead (section 9).
3. **No undervolting, overclocking, `ppfeaturemask` tweaks, unsupported register access, or driver-timeout disabling** unless a step below explicitly allows it and the owner approved it in writing in the run notes. A power cap *below* the default is allowed; **never raise the cap above `power1_cap_default`.**
4. **Crash budget:** the first intended power-cycle in a session is acceptable; **at most 2 machine crashes per session** without a new explicit owner approval. After any crash: stop, collect evidence, report, wait. **Never auto-resume a load after a reboot.**
5. After a crash the machine will probably boot Windows (default boot order) — the owner must press **F8** and choose the USB UEFI entry each time. Do not try to change boot order (no NVRAM writes).
6. Do not stop/restart/modify anything on the production server `192.168.1.13` or the Raspberry Pi `192.168.1.5` (it is the house router). Reading and copying explicitly named model files from `.13` (read-only `rsync`/`scp`) is allowed.
7. No secrets in logs or files. Do not copy private keys onto the target. Key-only SSH.
8. **Pin software versions for the whole campaign.** The kernel already changed once between sessions. Run `apt-mark hold` on `linux-image-*`, `mesa-*`, `libdrm*`, `firmware-amd-graphics` after the baseline, and record versions in every run.
9. The machine continuing to run is **not** a pass. A pass needs correct tool output, no kernel GPU errors, and telemetry inside limits.
10. If anything is ambiguous about a disk, a boot step or an approval: stop and ask the owner. Do not guess.

---

## 3. Architecture

```
controller (your computer) ──ssh──► target rx6900xt-test (Debian 13, user "rxtest", key-only)
   ├─ rxt (launcher, you write it)         ├─ /usr/local/sbin/rxt-run       (wrapper: marker, limits, restore)
   ├─ profiles/*.json (test definitions)   ├─ /usr/local/sbin/rxt-sampler   (telemetry, 10 Hz, stdout CSV)
   ├─ results/<date>/<run_id>/             ├─ /usr/local/sbin/gpu_envelope.sh (reference; see 3.3)
   └─ netconsole/journal receivers         └─ /var/lib/rxtest/ (state, markers, local copies)
```

### 3.1 Controller launcher (`rxt`)
One small launcher with subcommands (Python 3 stdlib + `ssh`/`rsync` is enough; no framework):
`discover`, `setup`, `inspect`, `run <profile>`, `status`, `stop`, `collect`, `recover`, `report`.
- SSH options: `-o BatchMode=yes -o ConnectTimeout=5 -o ServerAliveInterval=2 -o ServerAliveCountMax=3`; pin and verify the host key (record the fingerprint in `network.json`; do not disable host-key checking).
- `discover`: the target IP is unknown. Try, in order: the owner-supplied IP; `rx6900xt-test.local` (mDNS) / `rx6900xt-test` (router DNS); ARP/neighbour scan restricted to `192.168.1.0/24` filtered by the NIC MAC prefix seen in the logs. Never scan other subnets.
- Record per connection: controller UTC, `ssh target 'date +%s.%N; cat /proc/uptime; cat /proc/sys/kernel/random/boot_id'`. Loss of SSH + a changed `boot_id` after reconnect = machine restart.

### 3.2 Evidence that survives a power cut (do this *before* any R2 test)
A local file on the target loses the last seconds. Therefore:
- **Live streams to the controller** (each into the run folder, line-buffered, `fsync` on the controller side every second):
  1. telemetry: `ssh target 'sudo rxt-sampler --hz 10'` (spec in 3.4);
  2. kernel log: `ssh target 'sudo journalctl -k -f -o short-monotonic'`;
  3. **netconsole** (UDP, wired NIC): on target `modprobe netconsole netconsole=@/enp6s0,6666@<controller_ip>/<controller_mac>` (verify exact syntax with `modinfo netconsole` and kernel docs) and on the controller a UDP listener on port 6666 writing `netconsole.log`. This can carry the last kernel messages even when the disk write is lost. Verify it works with `echo test > /dev/kmsg` before relying on it.
- **Journald on the target:** `/etc/systemd/journald.conf.d/rxtest.conf` with `Storage=persistent`, `SyncIntervalSec=1s`, `RateLimitIntervalSec=0`; `mkdir -p /var/log/journal`; restart journald; confirm `journalctl --disk-usage` and `--list-boots`.
- Active-run marker `/var/lib/rxtest/active_run.json` written and `sync`ed **before** the load starts; removed only after clean completion. A marker found at the next boot means INTERRUPTED, never PASSED.

### 3.3 Reference script already on Drive
`RTX_Ryzen9/rx6900xt_usb/linux/gpu_envelope.sh` (`info`, `status`, `restore`, `run --label --minutes [--cap-pct] [--sclk-max] [--tj-stop] -- CMD`). It was tested **only against a simulated sysfs, never on the card**. Known limitations you should fix or replace: 1 Hz sampling of the smoothed `power1_average`; telemetry written locally without `fsync` (loses the last seconds); no streaming to the controller; no `pp_dpm_sclk` mask mode (uses overdrive, which needs a kernel parameter); `--cap-pct` is a percentage of `power1_cap_default`; the `timeout` wrapper does not kill process groups. Sysfs labels confirmed in the logs: hwmon temps are `edge`, `junction`, `mem`; `PPT`; `fan1`; `pwm1`. Treat the script as a starting point, not a requirement.

### 3.4 Telemetry sampler (`rxt-sampler`)
Output CSV to stdout, header first, **empty cell = sensor missing (never 0)**. Default 10 Hz; `--hz 1` for long soaks.
Columns: `t_mono_s,utc,boot_id,run_id,gpu_busy_pct,mem_busy_pct,power_avg_w,power_cap_w,vddgfx_raw,sclk_mhz,mclk_mhz,edge_c,junction_c,mem_c,fan_rpm,pwm,vram_used_mib`.
Sources (verify each exists on this kernel): `/sys/class/drm/card*/device/{gpu_busy_percent,mem_busy_percent,mem_info_vram_used,pp_dpm_sclk,pp_dpm_mclk,power_dpm_force_performance_level}` and `.../hwmon/hwmon*/{power1_average,power1_cap,power1_cap_default,power1_cap_min,power1_cap_max,freq1_input,freq2_input,temp1_input,temp2_input,temp3_input,temp*_label,fan1_input,pwm1,in0_input}`. The idle `vddgfx: 6.00 mV` in `sensors` is implausible: store the raw value and flag its units as unverified.
Optional second source: `amdgpu_top --json -s 100 -n 0` (documented flags: `--json/-J`, `-s <ms>`, `-n <count, 0 = forever>`, `-d -gm` for gpu_metrics, `--list`, `-i <idx>`, `--no-pc`). Its JSON schema is not documented in the README: inspect one sample before parsing and keep the raw JSONL. gpu_metrics may expose instantaneous socket power and voltages; use it only if it adds data the sysfs sampler lacks. Do not let sampling itself change the load (check `--no-pc` effect).

---

## 4. Result recording (keep it small, uniform, append-only)

```
results/<YYYY-MM-DD>/<run_id>/
  meta.json         # controller+target versions, profile, commands, cap, host/boot ids
  telemetry.csv     # from rxt-sampler (controller-side copy is the authoritative one)
  kernel.log        # journalctl -k -f stream (monotonic)
  netconsole.log
  stdout.log  stderr.log  command.txt
  summary.json      # verdict + metrics (schema below)
results/INDEX.jsonl     # one line per run (append-only)
results/RESULTS.md      # generated table from INDEX.jsonl; never hand-edited
results/crashes.jsonl   # one line per machine restart / abnormal end
```
`run_id` = `<UTC yyyymmddThhmmssZ>_<test_id>_<short-label>`.

`summary.json` fields: `run_id, test_id, class(R0|R1|R2), started_utc, ended_utc, duration_s, boot_id, kernel, mesa, llama_cpp_commit, tool_versions{}, cap_w, cap_pct_of_default, sclk_states, command, exit_code, verdict, stop_reason, samples, metrics{power_max_w,power_p95_w,junction_max_c,edge_max_c,mem_max_c,busy_mean_pct,vram_max_mib,throughput{...}}, kernel_errors[], output_validation{ok,detail}, notes`.

**Verdict enum (exactly one):** `PASS`, `FAIL_APP` (non-zero exit, OOM, wrong output), `FAIL_GPU` (amdgpu ring timeout/reset/fault in kernel log), `CRASH_MACHINE` (SSH lost + `boot_id` changed or uptime reset), `ABORTED_LIMIT` (your thermal/power stop), `ABORTED_OPERATOR`, `INVALID` (output validation failed), `SKIPPED` (state why), `NO_DATA`.

**Where the knowledge goes (avoid contradicting copies):**
- Raw data and `RESULTS.md` live under `RTX_Ryzen9/rx6900xt_usb/results/` (copy the controller's folder there when you have access to the Drive folder; otherwise hand it to the owner).
- In the dossier `AMD_Radeon_RX_6900_Linux_tests.md`, add **one** section `3.3 Session 2` with: one table row per run (`run_id | test | cap | duration | verdict | key metric | path`) and at most ten lines of interpretation, each marked `[measured]` or `[hypothesis]`. **Replace** any dossier statement your data contradicts — do not append a conflicting one. Update the hypothesis table in section 4 and the group table in section 5 accordingly. Do not touch other files except via section 8 (Phase 0).
- Facts only. Do not write recommendations that oblige a reader to act; write what happened, with numbers and run IDs.

---

## 5. Test catalogue

Risk classes: **R0** no GPU load. **R1** GPU load of a kind that already passed. **R2** GPU compute of the kind that tripped the machine; may power-cycle it. R2 requires: owner present at the keyboard/F8 (create a file `ARMED` on the controller or pass `--owner-present`), `sync` done, markers armed, streams running (3.2), crash budget not exhausted.
Always start each test with a short idle capture (10 s) and end with 60 s of cool-down capture.

### Phase A — baseline and forensics (R0, ~1.5 h)
- **A1 Inventory.** `uname -a`, `cat /proc/cmdline`, `lscpu`, `dmidecode -t bios,baseboard` (read-only), `lspci -nnvvv -s 09:00.0` (as root; keep Link status, `Resizable BAR` capability, AER counters), `lspci -vv -s 00:03.1` / the root port above the GPU, `vulkaninfo --summary`, `vainfo` (install `vainfo`, `mesa-va-drivers`), `ls /sys/class/drm/card*/device/`, dump of every hwmon file (names + values), `cat pp_dpm_sclk pp_dpm_mclk pp_power_profile_mode power_dpm_force_performance_level`, `pp_od_clk_voltage` if present, `amdgpu_top -d` / `-d -gm`, `/sys/kernel/debug/dri/*/amdgpu_firmware_info` (needs debugfs). Record `power1_cap_min/max/default` — they define the ladder.
- **A2 Find the crash evidence that exists.** `journalctl --list-boots`; for every boot print first/last timestamps and the last 40 kernel lines; classify each boot end as *clean* (power-off/reboot target reached) or *abrupt*. Extract the abrupt ones into `forensics/boot_<n>.log`. If the journal was not persistent in earlier sessions, say so. Also read-only check the ESP dirty bit (`fsck.vfat -n` on the ESP device, **not** `-a/-y`) and `dmesg | grep -i 'not properly unmounted'`.
- **A3 SSH audit.** `sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|pubkeyauthentication|allowusers)'`. The file `setup_rxtest.sh` appended `PermitRootLogin yes` and `PasswordAuthentication yes`; if effective, first confirm key login works from a **second** session, then fix with a drop-in `/etc/ssh/sshd_config.d/00-rxtest.conf` (`PermitRootLogin no`, `PasswordAuthentication no`) — Debian's `Include` is at the top, and the first value wins — and re-run `sshd -T`. Create a sudoers entry limited to the `rxt-*` helpers for user `rxtest`.
- **A4 Tool install (pinned).** Packages: `pciutils lm-sensors vulkan-tools mesa-vulkan-drivers vainfo mesa-va-drivers ffmpeg stress-ng jq git cmake build-essential libvulkan-dev glslc spirv-headers libcurl4-openssl-dev rsync python3`. `llama.cpp`: clone, **record the commit**, `cmake -B build -DGGML_VULKAN=ON -DCMAKE_BUILD_TYPE=Release && cmake --build build -j`, then `build/bin/llama-bench --list-devices` and confirm `Vulkan0` is the RADV 6900 XT and not `llvmpipe`. `memtest_vulkan`: use the **v0.5.0 release binary** (Rust project; do **not** build it with CMake) or `cargo build` at the pinned tag; record sha256 and URL (authenticity is not independently verifiable — say so). `amdgpu_top`: pinned release `.deb` from upstream releases or `cargo install --locked`, record version. `vkpeak` (nihui/vkpeak, MIT): prebuilt release or CMake build with submodules; usage is `vkpeak <device_id>`. FurMark is **not** installed by default (needs a display).
- **A5 Idle baseline.** 10 min at 10 Hz. Confirm label mapping, units, that fan stays 0 RPM at idle, and the sampler's own overhead.
- **A6 Evidence-channel check.** Prove the three streams (telemetry, journal -f, netconsole) arrive on the controller, then deliberately `echo "RXT-LINK-TEST" > /dev/kmsg`. Test `rxt recover` logic by killing the SSH session (not the machine).
- **A7 Platform control (R0 for the GPU, heavy for the PSU).** `stress-ng --cpu $(nproc) --cpu-method matrixprod --timeout 10m --metrics-brief`, then `stress-ng --vm 4 --vm-bytes 60% --timeout 5m`, with GPU idle. Record CPU temps/power (`sensors`, `turbostat` if available). Purpose: show whether the PSU/board survive CPU+RAM stress alone (if the machine restarts here, the GPU is not the common factor).

### Phase B — loads that previously passed or are low-risk (R1, ~2 h)
- **B1 VRAM test, default cap.** `memtest_vulkan` (no size argument exists). It shows a device menu when `llvmpipe` is present: type `1` (RADV) or let it auto-select after ~10 s. Run with `timeout --signal=INT 380 …`. PASS requires the final `no any errors, testing PASSed`. Repeat once at a 60 % cap (set by the wrapper). Note the step in the first session where one 30-s window wrote only 263 GB/s; log any recurrence.
- **B2 Light 3D (headless).** Check `vkmark --help` / `vkmark --list-winsys` for a headless or KMS winsys; if no usable surface, mark `SKIPPED: no display` — do not fake it. 5 min if it runs. (The earlier "vkmark 13,966" claim has no log.)
- **B3 Video engine (VCN 3.0).** `vainfo` must list the profiles; AV1 is decode-only on RDNA 2 (Mesa ≥ 21.3), H.264/HEVC encode+decode. Generate a 1080p30 reference clip with software encoders (`ffmpeg -f lavfi -i testsrc2=...`). Tests: (a) HEVC and H.264 **decode** with `-hwaccel vaapi -hwaccel_output_format vaapi -f null -`, looped for 5 min each; (b) **encode** with `-vaapi_device /dev/dri/renderD128 -vf 'format=nv12,hwupload' -c:v hevc_vaapi -qp 24` and `h264_vaapi`, 5 min; do not set `-compression_level` below 32 without testing (an upstream ticket reports corrupt output at low values on radeonsi); (c) AV1 decode if the sample exists. Validate every output: `ffprobe` clean, frame count equals input, `-lavfi ssim` against the reference (record the value). Record fps, GPU busy, power and temps. Exit with PASS only if the output validates.
- **B4 Safe compute control.** Qwen2.5-0.5B Q4_K_M (the same file as before; compute and record its sha256). `llama-bench -m <gguf> -ngl 99 -dev Vulkan0 -p 512 -n 0 -r 60 -o jsonl` (about 30 s of continuous prefill; first run at a **60 % cap**). Purpose: a first steady-state power/voltage/clock profile of GEMM-type work with a model known to pass.
- **B5 Peak-shader probe (optional, R1→R2 boundary).** `vkpeak <device_id>` at a 60 % cap only. It is a synthetic peak-throughput tool; it is not part of the owner's workflow. Skip if B4 showed any instability.

### Phase C — power-cap ladder on the known trigger (R2, ~1.5 h, owner present)
The trigger is **Qwen2.5-3B Q4_K_M, pp1024**. Because the trip happened within seconds, short steps are informative.
- Command template (validate every flag on the installed version): `llama-bench -m <qwen2.5-3b-instruct-q4_k_m.gguf> -ngl 99 -dev Vulkan0 -p 1024 -n 0 -r <N> -o jsonl --progress` (flags confirmed in upstream `llama-bench` README: `-p`, `-n`, `-r`, `--delay`, `-ub`, `-b`, `-ngl`, `-dev`, `--list-devices`, `-o`, `-d`, `-pg`, `--progress`, `--no-warmup`).
- Ladder (percent of `power1_cap_default`, rounded to whole watts, clamped to `power1_cap_min..default`): **50 → 60 → 70 → 80 → 90 → 100 %**. At each level: **C-a** 60 s (about 30–60 repetitions), then, if PASS, **C-b** 5 min. Record throughput (pp1024 t/s) and power at each level — this gives the performance-per-watt curve even if nothing fails.
- Stop rules: first `CRASH_MACHINE` ends the ladder. Do not try the next higher cap after a crash. Do not repeat the crashing step. The highest PASS level (with C-b passed) is the *candidate envelope*, labelled as such — not a certificate.
- If **50 % fails**, that is a major finding (average power is not the discriminator): stop the ladder and report.
- Then, at the candidate envelope: **C-c** 15 min and **C-d** 60 min soak with the same command (loop with `-r` or a shell loop), still R2.

### Phase D — clock cap instead of power cap (R2, ~45 min)
Use the DPM level mask, which needs no kernel parameter: read `pp_dpm_sclk`; `echo manual > power_dpm_force_performance_level`; `echo "<allowed level indices>" > pp_dpm_sclk` to exclude the highest state(s); verify by re-reading; run the C-a step (60 s) at **default power cap**, then step the allowed highest level up one at a time. Always restore (`auto` and the full mask) and verify. Record the exact levels and measured sclk. (Overdrive `pp_od_clk_voltage` needs `amdgpu.ppfeaturemask`; do **not** change the kernel command line without owner approval — it is a reversible GRUB edit on the USB SSD only, but still a decision for the owner.)

### Phase E — load shape and size (R2; run only after C/D, at the candidate envelope first)
- **E1 Batch ramp:** `-ub 32,64,128,256,512` with the 3B pp1024 command (`-ub` = physical micro-batch). Does a smaller micro-batch avoid the trip at default cap?
- **E2 Duty cycle:** repeated short bursts with idle gaps (a shell loop around `llama-bench -r 1 --no-warmup`; `--delay` only pauses between *tests*, not repetitions). Shapes: 1 s on / 2 s off, 5 s / 5 s, 10 s / 2 s for 10 min each.
- **E3 Model size threshold** at the candidate cap: 0.5B (known pass) → Qwen2.5-1.5B Q4_K_M → 3B. Download with recorded source URL + sha256.
- **E4 Attention/precision knobs** (only if time allows): `-fa on|off`, and the llama.cpp Vulkan environment toggles that exist in the installed source (`grep -rn "GGML_VK_" ggml/src/ggml-vulkan/`; the upstream build doc lists none). Change one variable at a time.
- **E5 Reproduce at default cap (deliberate crash).** Only if the owner approves in writing, the crash budget allows it, and C/D found an envelope: run the exact original sequence (`-p 512,1024 -n 128 -r 2`) at 100 %, to obtain **time-to-trip with streamed telemetry and netconsole**. This is the single most informative R2 step, and the only one whose purpose is a crash.

### Phase F — workloads the owner actually runs (R1 inside the validated envelope; skip for any profile that tripped)
Compare with known numbers from `orquestra/maquinas.json`: G14 RTX 3060 6 GB — Qwen3.5-4B 68 tok/s, prompt 2,100 tok/s; Qwen3-VL-8B 50 tok/s; Qwen3-Embedding-0.6B prompt 13,000 tok/s; server CPU baselines in the same file. Copy models **read-only** from `192.168.1.13` (ask the owner for the path; do not browse production directories); verify sha256 on both sides.
- **F1** Qwen3.5-4B Q4_K_XL: `llama-bench -p 512,1024 -n 128 -r 5 -o jsonl` and one `llama-server` run for 10 min at light load.
- **F2** Qwen3-Embedding-0.6B: embeddings throughput (`llama-bench -embd 1` as supported by the installed build, or `llama-embedding`).
- **F3** Vision (Qwen3-VL-8B + mmproj): `llama-mtmd-cli` with 20 owner-provided or generated images; record time per image and VRAM; validate outputs by checking that the model returns non-empty, schema-valid text.
- **F4** whisper.cpp with the Vulkan backend (`large-v3-turbo` or smaller) on a 10-min sample: real-time factor and WER against a known transcript if one is available.
- **F5** Combined duty-cycle soak: 60 min mixing F1/F2 requests with idle gaps, at the validated cap. Output validity checked, not just uptime.
For each: tokens/s or items/s, **watts, joules per item, VRAM, and the same numbers at 2–3 cap levels** (a performance-per-watt table).

---

## 6. Run procedure (every R1/R2 run)
1. `rxt status` — no marker, correct `boot_id`, versions unchanged, free disk space, streams up.
2. For R2: confirm `ARMED`, crash budget, owner present; announce the step in the run notes.
3. Wrapper sets the cap/mask, readback-verifies, writes the marker, starts the load with a hard time limit.
4. Controller streams telemetry/kernel/netconsole; evaluates stop rules (junction ≥ 95 °C, edge ≥ 90 °C, mem ≥ 95 °C, loss of telemetry for > 3 s while SSH is alive, kernel `amdgpu` error line).
5. On completion: restore settings, readback-verify, collect `stdout/stderr`, run the output validator, write `summary.json`, update `INDEX.jsonl`, regenerate `RESULTS.md`, delete the marker.
6. If SSH drops: wait up to 3 min for the host; if it returns with a new `boot_id` → `CRASH_MACHINE`; collect `journalctl -b -1 -k` and `-b -1` tail, `last -x`, the ESP dirty check; append to `crashes.jsonl`; **stop and report to the owner**. If it does not return, tell the owner exactly: "press F8 and select the USB UEFI entry".

## 7. Time and order
Phase A (1.5 h) → B (2 h) → C (1.5 h) → D (45 min) → E (1.5 h) → F (2–3 h). Stop at the first condition that changes the picture (for example, 50 % cap failing). Report after each phase with the table row(s) only.

## 8. Phase 0 — documentation fixes
Before testing, apply the corrections listed in `RTX_Ryzen9/rx6900xt_usb/PHASE0_DOC_FIXES_2026-10-15.md` (exact file, old text → new text). They are local edits on the Drive-synced folder; make them with ordinary text-edit tools. Do not rewrite whole files, do not change line endings, and do not touch anything under `_ARQUIVO*` or the owner's backup folder (`My Drive\IT\backup\`). Re-read each target immediately before editing: other agents have edited these files today.

## 9. Owner-only tests (cannot be automated; request them with the information they give)
| ID | Action | What it separates | Notes |
|---|---|---|---|
| G1 | Put the card in the production server `.13` (different PSU) or another PC and repeat Phase C there | board vs PSU/platform | **Highest information.** Needs a free slot and 2×8-pin. Requires physical work and an OS with Vulkan |
| G2 | Move each 8-pin lead to different PSU output sockets; reseat the card | cable/socket/contact | cheap; owner states two separate cables already |
| G3 | BIOS: enable Resizable BAR/Above 4G Decoding (BAR is 256 MB now) and repeat C-a at 100 % — and/or force PCIe Gen3 on the x16 slot | PCIe / BAR state | BIOS changes need explicit owner approval and a note of the original values |
| G4 | Put another GPU in this PC and run Phase C's platform controls | platform vs card | optional |
| — | **VBIOS flash with another vendor's ROM (Plan D of the 2026-07-21 research report) — NOT recommended** | — | See `AMD_Radeon_RX_6900_Linux_tests.md` §6: it needs an SSID-mismatch force flag, can brick the card, voids what is left of the warranty, and nothing in the evidence points to the VBIOS as the cause (the card has an unproven dual-BIOS switch). The report that proposes it is unsourced on this point. Do not execute or re-propose it unless a measured result specifically implicates the firmware. |

## 10. Final deliverables
- `results/` tree as in section 4, copied to `RTX_Ryzen9/rx6900xt_usb/results/`.
- The dossier updated per section 4, with corrections *replacing* wrong text.
- A short handoff to the owner: what ran, verdict table, crash events, the candidate envelope (if any) with its limits, what remains manual (section 9), and the exact commands to repeat the best profile.
- Do not claim hardware health, stability or a "fix". Say what was measured, for how long, under which cap and software versions.
