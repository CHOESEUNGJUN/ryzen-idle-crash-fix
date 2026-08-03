*[한국어](README.ko.md)*

# Ryzen 5000-series (5900X) random shutdown/reboot while idle — on a 24/7 system (home server, Proxmox, NAS)

> **Disclaimer:** None of this has been confirmed with an oscilloscope or any real electrical measurement — everything below is inferred from BIOS settings, software sensor logging, and crash-free uptime. This exact fix may not solve it for you. It solved it for me.

## Symptom

A Ryzen 5000-series CPU (this was reproduced on a 5900X + ASUS ROG Strix X570-E Gaming, but the mechanism appears to affect the wider Ryzen 2000–9000 / AM4–AM5 lineup based on community reports, across other vendors/boards too) randomly shuts down or reboots **only while idle or at low load** — never under sustained heavy load. No overheating, no obvious error in the OS logs beforehand. If you have a hardware watchdog enabled, you may just see a watchdog-triggered reset with no other explanation. If you catch a machine-check exception (MCE) in the logs, it typically decodes to a core-internal bank with no memory address captured (`ADDRV=0`), which rules out a RAM/DIMM fault even though it can superficially look memory-related.

This is easy to miss on a desktop that's rarely fully idle (gaming, browsing), but it shows up constantly on a system that's designed to sit idle for long stretches — a home server, hypervisor host, NAS, or anything meant to run unattended 24/7.

## What it is NOT

Ruled out through direct testing, in case you're chasing the same ghost:
- **RAM** — multiple full passes of memtest came back clean.
- **A single bad CPU core** — isolating/offlining one specific physical core (the one implicated in the one MCE event that did occur) did **not** stop the crashes; several more crashes happened afterward with zero repeat MCEs. If you're tempted to blame "one bad core," get more evidence first — it's very likely a red herring.
- **The motherboard or PSU** — no direct evidence either way, but the fix below addresses this at the CPU/VRM firmware level, not a hardware fault.

## My working hypothesis

This looks like a transient current/voltage handling issue during idle-state power transitions, not a static "voltage too low" problem you'd normally associate with manual undervolting. In other words: **it behaves exactly like a failed undervolt, even if you never touched voltage settings yourself** — because the stock/BIOS-default idle power management on some boards apparently doesn't give the VRM enough current headroom for the very brief transient spike that happens when the CPU snaps from a deep idle state back to an active one (or vice versa). My best read is that this is a BIOS/firmware-level current-limit configuration issue (the CPU's "digital fuse" being set too conservatively for this specific transient), not a hardware defect — but I want to be upfront that the actual microsecond-scale VRM transient can't be directly observed with any software sensor polling (even fast polling only samples ~once per second), so this is a strong correlation from a fix that worked, not a fully proven root cause.

## What did NOT fix it

- Disabling Global C-states and setting "Power Supply Idle Control" to a less aggressive idle mode, alone, did not stop the crashes.
- A positive **VDDCR CPU Offset** (a flat voltage curve shift) barely moved the actual idle voltage floor at all (in logged testing, less than 0.02V), while pushing the boost-clock voltage ceiling well past AMD's own documented ~1.55V hard cap for this generation — i.e. it added real risk without addressing the actual problem. If your board exposes a separate Offset vs Fixed/Manual voltage mode, be aware they behave very differently — the idle floor and the boost ceiling are not simply "the same curve shifted."

## What did fix it (so far)

Applied together, in the BIOS, all under the CPU/PBO section (menu names vary by vendor):
1. **Precision Boost Overdrive (PBO): Manual** (not Auto)
2. **EDC (Electrical Design Current) limit: raised** — this is the key one. The stock/default EDC limit appears too small for the actual transient current draw of this CPU generation during a power-state transition. Raise it meaningfully (double the default in this case).
3. **PPT (total power) limit: raised** to a level your cooling can sustain.
4. **Platform/CPU thermal throttle limit: set manually**, comfortably below the CPU's real thermal limit.
5. A small supplementary **VDDCR CPU Offset** (+0.1V in this case) — kept as a minor nudge on top of the above, not the primary fix.

Cooling used here: a dual-tower air cooler (DeepCool AK-series, roughly). The exact PPT/thermal-limit numbers above are only sane with cooling in that class or better — treat the *existence* of these settings as the takeaway, not the specific values, if your cooler is smaller.

Result so far: direct voltage sampling shows the idle voltage floor now settles in a stable, healthy range and does **not** dip toward the low value (~0.86V) that was directly observed right before an earlier crash. The system has now run well past its previous best-ever crash-free uptime with zero crashes, and is being watched for a much longer stretch (hundreds of hours) before calling it conclusively fixed.

## If you're debugging this yourself

- Log CPU core voltage, temperature, and load continuously (even a simple 1-minute-interval script logging your platform's sensor output is enough) so you have data from right before the next crash, not just after.
- Check machine-check-exception decode (if any) for the bank number and whether an address was captured — no address strongly suggests not a memory fault.
- Don't jump to "bad core" from one MCE event — test it, don't assume it.
- If you try a voltage offset first, measure the actual idle floor and boost ceiling before/after — don't assume it did what you wanted.
- Look specifically at your BIOS's PBO/current-limit (EDC/TDC/PPT) settings before spending more time on voltage-curve settings — for this specific CPU/idle-crash signature, the current limit was the missing piece, not the voltage curve.

*Written up in the hope it saves someone else the multi-week debugging loop this took.*
