*[한국어](README.ko.md)*

# Ryzen 5000-series (5900X) random shutdown/reboot while idle — on a 24/7 system (home server, Proxmox, NAS)

## 1. Overview

A record of a random shutdown/reboot occurring only in the idle state on a Ryzen 5000-series CPU in a 24/7 system (home server, Proxmox, NAS).[^1]

## 2. Symptom

- Occurs only **while idle or at low load**, never under sustained heavy load.
- No overheating. No advance warning in the logs.
- If a hardware watchdog is enabled, only a watchdog-triggered reset is logged, often with no further explanation.
- When a machine-check exception (MCE) is captured, it typically decodes to a core-internal bank with no memory address recorded (`ADDRV=0`) — superficially memory-like, but not an actual RAM fault.
- Easy to miss on a desktop that is rarely fully idle (gaming, browsing); shows up consistently on a system designed to sit idle for long, unattended stretches.

## 3. Ruled-out causes

- **RAM** — multiple full memtest passes came back clean.
- **A single bad CPU core** — isolating/offlining the physical core implicated in the one MCE event did not stop the crashes; several more crashes occurred afterward with zero repeat MCEs. A single MCE event is insufficient grounds to conclude a core fault.
- **Motherboard or PSU** — no direct evidence either way. Not separately verified, since the fix below addresses this at the CPU/VRM firmware-configuration level rather than a hardware fault.

## 4. Root-cause analysis

This is assessed as a transient current-handling issue during idle↔active power-state transitions, not a static "voltage too low" condition.[^2] It reproduces the same behavior as a failed undervolt even without ever touching voltage settings directly. The stock BIOS-default idle power management on some boards appears not to provide sufficient current headroom for the brief transient spike at this transition.

In short: the root cause is assessed as the **current-limit (EDC) configuration**, not the voltage curve. This has not been confirmed with direct electrical measurement (e.g. oscilloscope); it is inferred from the outcome of the applied fix.

## 5. Actions taken

### 5.1 Ineffective

- Disabling Global C-states + setting Power Supply Idle Control to Typical mode — did not stop the crashes on its own.
- A positive VDDCR CPU Offset — measured idle voltage floor barely moved (<0.02V), while the boost-clock voltage ceiling exceeded AMD's documented ~1.55V hard cap for this generation. Added risk without resolving the issue.

### 5.2 Applied fix

Applied together, in the BIOS, under the CPU/PBO section:

1. Precision Boost Overdrive (PBO): **Manual** (not Auto)
2. EDC (Electrical Design Current) limit: **raised** — key change. The default limit is assessed as too low for this CPU generation's transient current demand during a power-state transition; raised to roughly double the default.
3. PPT (total power) limit: raised to the level the cooling can sustain.
4. Platform/CPU thermal throttle limit: set manually, with margin below the real thermal limit.
5. VDDCR CPU Offset +0.1V — supplementary, not the primary fix.

Cooling used: a dual-tower air cooler (DeepCool AK-series). The PPT/thermal-limit values above assume cooling of that class or better; with a smaller cooler, treat the existence of these settings as the takeaway rather than the specific numbers.

The EDC/PPT/PBO-Manual direction itself is based on a community report of the same CPU/motherboard combination hitting the same symptom. The contribution of this document is applying and tuning those values for this specific setup, with before/after voltage and temperature measurement.

## 6. Result

200+ hours (8+ days) of continuous, crash-free uptime confirmed after the fix. Direct voltage sampling shows the idle voltage floor holding in a stable range, with no recurrence of the low value (~0.86V) observed immediately before an earlier crash. Assessed as resolved.

## 7. Notes for debugging the same issue

- Log voltage, temperature, and load continuously — data from immediately before a crash is required, not just after.
- On an MCE, check whether the bank number and address were captured. No address captured suggests a memory fault is unlikely.
- Do not conclude a specific core is at fault from a single MCE event without further testing.
- If trying a voltage offset first, measure the actual idle floor and boost ceiling before and after — do not assume it did what was intended.
- Check the BIOS's PBO/current-limit settings (EDC/TDC/PPT) before spending more time on the voltage curve.

If you're seeing the same symptom, or have results from a different CPU/board/cooling combination, please open an issue.

[^1]: Reproduced on a 5900X + ASUS ROG Strix X570-E Gaming combination. Community reports suggest the mechanism affects the wider Ryzen 2000–9000 / AM4–AM5 lineup, across vendors and boards.
[^2]: The microsecond-scale VRM transient cannot be directly observed via software sensor polling (roughly once per second at best). This root-cause analysis is therefore a strong correlation from a fix that worked, not a root cause confirmed by direct electrical measurement.
