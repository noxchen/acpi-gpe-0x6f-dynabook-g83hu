# dynabook G83/HU (Tiger Lake): ACPI GPE 0x6F storm

The firmware of this laptop asserts ACPI **GPE 0x6F** continuously (~340
interrupts/second) from boot. Every event runs an OEM AML chain
(`\_GPE._L6F` → `\_GPE.AL6F` → `\_SB.ISME/GNMB/GGRP/GADR/GINF`), which costs
~2.8 ms per event, so one CPU core is permanently pinned in the ACPICA
interpreter.

Symptoms: the fan never spins down, battery drains, and `s2idle` is unusable
(the machine wakes ~2.2 s after every idle suspend).

Verified fix: `acpi_mask_gpe=0x6F` on the kernel command line.  With it:
package power 14.8 W → 2.9–3.4 W, package temp 60 °C → 39–44 °C, cpu0 system
time 98 % → 0–1 %, and the machine sleeps normally again.

## Contents

| File | Description |
|---|---|
| `upstream-report.txt` | full bug report, plain text (paste into bugzilla / LKML) |
| `dsdt-g83hu.aml` | the machine's DSDT (`/sys/firmware/acpi/tables/DSDT`, 464514 bytes) |
| `dmesg.log` | boot dmesg (MAC addresses, UUIDs and private IPs redacted) |
| `interrupts.txt` | `/proc/interrupts` snapshot while the storm was running |

---


## Environment

| | |
|---|---|
| Machine | Dynabook Inc. dynabook G83/HU, model `A6GHHUBAD515` |
| Board | `A010D/0001` |
| BIOS | Version **7.80**, 05/09/2022, vendor "Dynabook Inc." (AMI based) |
| CPU | 11th Gen Intel Core i5-1145G7 (Tiger Lake-UP3, 4C/8T) |
| Kernel | `7.2.15-1-omarchy` (Arch / Omarchy), ACPICA 20260408, `PREEMPT_DYNAMIC`, 8 CPUs |
| Sleep | `s2idle` only (`/sys/power/mem_sleep` = `[s2idle] deep`) |

## Summary

From the very first second after boot, ACPI **GPE 0x6F fires continuously at
~340 interrupts/second**.  Every event evaluates the AML method
`\_GPE._L6F`, which forwards to `\_GPE.AL6F` and from there evaluates the OEM
helpers `\_SB.ISME`, `\_SB.GNMB`, `\_SB.GGRP`, `\_SB.GADR` and `\_SB.GINF`
(these live next to the `ACPI000E` device `\_SB.AWAC`).

The AML interpreter execution costs roughly 2.8 ms per event, so one CPU core
is permanently pinned in kernel space:

* `cpu0`: **97–98 % system time**, ~1 % idle (the other 7 cores are idle)
* `kworker/0:1+kacpid` ~37 %, `kworker/0:3-kacpid` ~26 %,
  `kworker/0:0+kacpi_notify` ~21 % (lifetime averages)
* package power: **14.0–14.9 W** with the storm, **5.4 W** without
* package temperature: **59–61 °C** with the storm, **41–44 °C** without
* visible symptom for the user: the fan never spins down, battery drains fast

In addition the storm makes `s2idle` useless: the machine wakes up ~2 s after
every idle suspend (`/sys/power/pm_wakeup_irq` = `9`, i.e. the shared ACPI
IRQ), the firmware re-enables GPE 0x6F on every resume, and the cycle repeats
every ~30 s (65 suspend/resume cycles recorded in one 30 minute window).

## Evidence

### 1. The GPE counter

```
$ cat /sys/firmware/acpi/interrupts/gpe6F
15530228     STS enabled      unmasked      <- ~13.5 h uptime, 15.5 M events
$ cat /sys/firmware/acpi/interrupts/gpe_all
16586362

$ cat /sys/firmware/acpi/interrupts/gpe6E
 1061573  EN     enabled      unmasked      <- the neighbouring/EC GPE, ~21/s: normal
```

Sampled over 10 s: `gpe6F` +3437 (+344/s), `gpe_all` +3644 (+364/s).

```
$ grep '^\s*9:' /proc/interrupts
   9:   17416911          0          0  ...  IR-IO-APIC  9-fasteoi   acpi
```

`acpi_mask_gpe` / the sysfs knob both stop it:

```
$ echo disable > /sys/firmware/acpi/interrupts/gpe6F
# gpe6F stops counting, ACPI IRQ rate 367/s -> 21/s (gpe6E only)
```

### 2. The CPU time is spent inside ACPICA, not in any driver

`ftrace` function tracer on the busy kworkers (`kworker/0:*`), 2 s window,
top entries:

```
   4416 acpi_ut_track_stack_ptr
   4348 _raw_spin_lock
   4342 acpi_os_acquire_lock
   4330 acpi_ut_update_ref_count.part.0
   2852 acpi_ut_status_exit
   2487 acpi_ut_trace_ptr
   1574 acpi_ps_get_opcode_info
    962 kmem_cache_alloc_noprof
    805 acpi_ut_exit
    666 acpi_ut_create_generic_state
    268 acpi_ut_valid_name_char
    252 acpi_ps_next_parse_state
```

Caller chain is entirely `acpi_ps_parse_loop()` / `acpi_ps_create_op()` /
`acpi_ex_resolve_operands()` — i.e. AML bytecode being interpreted in a loop.
No device driver `->notify()` handler is ever reached, so no subsystem is
actually consuming the events.

### 3. Which AML methods run

Using the built-in ACPICA method tracing:

```
$ echo 0x10 > /sys/module/acpi/parameters/trace_debug_level   # ACPI_LV_TRACE_POINT
$ echo 0x80 > /sys/module/acpi/parameters/trace_debug_layer   # ACPI_TRACING
$ echo enable > /sys/module/acpi/parameters/trace_state
$ dmesg | grep -oE 'Method (Begin|End) \[[^]]+\]' | awk '{print $3}' | sort | uniq -c | sort -rn
    476 [\_SB.GINF]
    237 [\_SB.GADR]
    118 [\_SB.ISME]
    118 [\_SB.GNMB]
    118 [\_SB.GGRP]
     30 [\_GPE._L6F]
     30 [\_GPE.AL6F]
```

(0.4 s window; the tracing itself slows the storm down from ~344/s to ~75/s,
hence only 30 `_L6F` entries.)

So the chain is:

```
GPE 0x6F  ->  \_GPE._L6F  ->  \_GPE.AL6F  ->  \_SB.ISME / GNMB / GGRP
                                          ->  \_SB.GADR -> \_SB.GINF
```

`_L6F` also references `\_SB.AWAC` (the `ACPI000E` device, currently unbound
in Linux).  The `GNMB`/`GGRP` helpers look like NVM/group reads on top of an
SMBus/OPREGION, which is why a single invocation takes milliseconds:

```
$ grep -c 'Method End' dmesg   # 30 'Method End [\_GPE._L6F]' in 0.4 s
   -> ~2.8 ms of CPU time per GPE event
```

### 4. Power / temperature delta

```
storm running : package 14.0 W,  Package id 0 +60 °C, cpu0 sys 98 %
echo disable  : package  5.4 W,  Package id 0 +42 °C, cpu0 sys  6 %   (25 s later)
echo enable   : package 14.9 W,  Package id 0 +57 °C, cpu0 sys 97 %   (15 s later)
```

`/sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw` = 28 W, so the
storm alone eats ~1/3 of the package power budget while the machine is idle.

### 5. s2idle never survives

```
$ sudo dmesg | grep -E 'PM: suspend (entry|exit)'
[41801.247917] PM: suspend exit
[41829.215378] PM: suspend entry (s2idle)
[41831.478825] PM: suspend exit        <- 2.2 s later
[41859.464678] PM: suspend entry (s2idle)
[41861.760777] PM: suspend exit
... (65 such cycles in ~30 min)

$ cat /sys/power/pm_wakeup_irq
9                                       <- the ACPI IRQ woke the system

$ cat /sys/firmware/acpi/interrupts/gpe6F     # after each resume
16297917     STS disabled      unmasked        <- got re-armed by the firmware
16301579     STS enabled       unmasked
```

Note also that writing `disable` into `.../interrupts/gpe6F` returns `EINVAL`
for a few seconds right after resume: at that point the GPE is still
"disabled" from the ACPI core point of view (so `acpi_disable_gpe()` fails),
and a moment later the firmware/hardware asserts it again and it becomes
enabled.

### 6. Unrelated but probably the same firmware quality: a broken EC query

```
[48046.513805] ACPI BIOS Error (bug): Could not resolve symbol [\_SB.PC00.LPCB.EC._Q01.CHRG], AE_NOT_FOUND (20260408/psargs-365)
[48046.513817] ACPI Error: Aborting method \_SB.PC00.LPCB.EC._Q01 due to previous error (AE_NOT_FOUND) (20260408/psparse-543)
```

`_Q01` in the DSDT references a `CHRG` object that does not exist, so the
whole EC query handler aborts every time the EC raises query 0x01.

## Reproduce

1. Boot the machine (nothing needs to be plugged in, no special workload).
2. `watch -n1 'cat /sys/firmware/acpi/interrupts/gpe6F'` → increments ~340/s.
3. `top` → `kworker/0:1+kacpid`, `kworker/0:3-kacpid`, `irq/9-acpi` on CPU 0;
   `mpstat -P 0` → ~98 % `%sys`.
4. `echo disable | sudo tee /sys/firmware/acpi/interrupts/gpe6F` → the storm
   stops immediately and package power drops by ~9 W.

## Workarounds that work

* Runtime: `echo disable > /sys/firmware/acpi/interrupts/gpe6F`
  (does **not** survive suspend/resume).
* Boot time: kernel parameter `acpi_mask_gpe=0x6F`
  (needs to be verified across suspend/resume on this box).
* A polling user-space watchdog that re-writes `disable` every 3 s.  This was
  the only reliable variant here: a oneshot unit ordered `After=suspend.target`
  that masks once right after resume is **not** enough, because the GPE is
  re-armed again roughly 10-12 s after the resume (observed: masked at
  T+3 s, back at T+14 s, i.e. two re-arms per suspend cycle).  With the
  watchdog the residual storm is < 3 s per cycle, package power 4 W,
  `cpu0` back to ~1 %.

Downside of masking: no AML handler runs any more, so whatever OEM event
GPE 0x6F was supposed to deliver is lost.  Nothing in Linux appears to use it
(no `->notify()` handler is ever invoked, the `ACPI000E` device is unbound),
and the event is clearly stuck in the firmware, so this seems acceptable.

## Update (verified on this machine): the hard mask fixes both symptoms

With `acpi_mask_gpe=0x6F` added to the command line and the machine
rebooted, without any user-space help:

```
$ cat /proc/cmdline | tr ' ' '\n' | grep acpi_mask_gpe
acpi_mask_gpe=0x6F

$ cat /sys/firmware/acpi/interrupts/gpe6F
       0     STS disabled     masked        <- count 0 since boot, "masked" = hard mask

$ grep '^\s*9:' /proc/interrupts          # 5 s delta -> 21/s (was 350/s)
   9:       5033          0  ...  IR-IO-APIC  9-fasteoi   acpi

cpu0:   sys 0-1 %, idle 94-97 %            (was sys 97-98 %)
package power: 2.9-3.4 W                   (was 14.0-14.9 W)
package temp:  39-44 °C                    (was 59-61 °C)
```

And the s2idle problem disappears with it:

```
[  185.778749] PM: suspend entry (s2idle)
[  207.251253] PM: suspend exit            <- 21.5 s, RTC alarm
[  235.433080] PM: suspend entry (s2idle)
[ 1108.609446] PM: suspend exit            <- 873 s uninterrupted
[ 1136.684731] PM: suspend entry (s2idle)
[ 4239.343422] PM: suspend exit            <- 3102 s uninterrupted
```

Before the mask every single suspend exited again after ~2.2 s.  So the
storming GPE 0x6F was also what woke the system out of S0ix: the GPE keeps
firing while suspended and the shared ACPI IRQ aborts the s2idle entry.

Note that after a resume the state field reads `enabled masked`: the kernel
sets the runtime-enable bit again on resume, but the hard mask keeps the GPE
masked, so the counter stays at 0 and no AML is executed.

## Proposed fix

1. A DMI quirk (e.g. in `drivers/acpi/acpi_quirks` style table or via
   `acpi_mask_gpe` defaults) that masks GPE 0x6F for the dynabook G83/HU.
   This single change fixes both the CPU/power burn and the broken s2idle.
2. Or find out why the GPE status bit stays asserted: the firmware apparently
   never clears the underlying source, and the `_L6F`/`AL6F` handler only
   reads state (`GNMB`/`GGRP`/`GADR`/`GINF`) without acking anything.
3. The ACPI core could also treat this as a GPE storm automatically
   (`gpe_storm_threshold`), which does not seem to trigger here.

## Attachments

* `acpi-gpe-6f-DSDT.aml` — the machine's DSDT (copied from
  `/sys/firmware/acpi/tables/DSDT`, 464514 bytes)
* `acpidump -b` output for the full table set (please run on the affected
  machine if reproducing)
* `dmesg` from the affected boot
