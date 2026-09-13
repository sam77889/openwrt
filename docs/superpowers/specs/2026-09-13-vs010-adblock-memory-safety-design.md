# VS010 Adblock Memory Safety Design

**Date:** 2026-09-13

**Target:** Unicom VS010 (`qualcommax/ipq50xx`)

## Problem

The VS010 has 256 MiB of physical RAM, but only about 183 MiB is managed by
Linux after reserving memory for the kernel, boot firmware, and the 48 MiB WCSS
region used by the two Wi-Fi radios. A normal boot with SmartDNS,
HTTPS DNS Proxy, dnsmasq, and both radios leaves only about 8–12 MiB reported as
`MemAvailable`. The current image has no swap.

The default adblock configuration downloads three feeds into `/tmp`, which is a
tmpfs backed by RAM. During a reproduced reload, the downloaded and prepared
files grew `/tmp` from 96 KiB to more than 16 MiB. Adblock then started `sort`
with an 8 MiB buffer. `MemAvailable` reached zero, direct reclaim stalled the
serial shell for tens of seconds, and the kernel OOM killer killed `sort`.
Adblock subsequently continued with the next feed, prolonging memory pressure.

This is a bounded-memory processing failure rather than evidence of a
long-running userspace leak. The fix must nevertheless clean up orphaned
temporary state and fail safely so that interrupted updates cannot accumulate
RAM usage or corrupt the active blocklist.

## Goals

- Keep the existing three adblock feeds and the SmartDNS/HTTPS DNS Proxy setup.
- Prevent adblock updates from exhausting RAM on VS010.
- Stop an update immediately after a low-memory or sorting failure.
- Remove temporary download and processing files as soon as they are no longer
  needed and on normal or catchable abnormal exit.
- Preserve the last valid blocklist when an update fails.
- Add compressed swap headroom to the VS010 image.
- Provide automated regression tests for memory admission, cleanup, failure
  propagation, and image package selection.

## Non-goals

- Changing the VS010 Ethernet topology, WAN negotiation, or DTS port mapping.
- Reducing the WCSS reserved-memory region.
- Removing any configured adblock feed or DNS service.
- Moving high-churn adblock temporary files onto NAND flash.
- Claiming that zram alone fixes unbounded temporary-file behavior.

## Design

### 1. VS010 zram

Add `zram-swap` to `Device/unicom_vs010` in
`target/linux/qualcommax/image/ipq50xx.mk`. Its dependency selects `kmod-zram`
and the required BusyBox swap features. The existing OpenWrt zram service uses
half of `MemTotal` as the logical zram size when no explicit UCI size is set;
on this target that is about 89 MiB. Zram allocates compressed backing memory on
demand, so this does not reserve 89 MiB at boot.

No VS010-specific zram size or compression algorithm is introduced. This keeps
the target on the maintained OpenWrt defaults and avoids duplicating system UCI
configuration.

### 2. Memory admission control

Adblock gains an `adb_memreserve` setting with a default of 16 MiB. Available
processing headroom is calculated in MiB as:

```text
MemAvailable + SwapFree
```

Both values come from `/proc/meminfo`. Invalid or missing values fail closed.
The meminfo path is overridable only through the test environment so tests can
exercise deterministic memory states.

Admission checks run:

1. before an update changes or flushes the active DNS blocklist;
2. before each feed download starts;
3. after each download and immediately before sorting;
4. before merging all prepared feeds and before rendering the final backend
   file.

The pre-sort check requires the configured reserve plus the active sort-buffer
size. Other checks require the configured reserve. A rejected update logs the
phase, required MiB, and observed MiB, then exits without replacing the active
blocklist.

Using `MemAvailable + SwapFree` means the existing 183 MiB/no-swap VS010 refuses
an unsafe update, while the same device with zram can use compressed swap as
headroom. It also allows larger devices without swap to continue operating when
they genuinely have enough RAM.

### 3. Bounded temporary-file lifetime

After a feed has been converted into its sorted per-feed representation and its
compressed backup has been written, its raw download, category archive, and
intermediate concatenation files are removed immediately. They are not retained
until the end of the complete multi-feed update.

The update keeps only data required by a later merge. The existing common
temporary directory remains under `adb_basedir`; no writes are redirected to
flash.

### 4. Fail-fast propagation

Feed processing currently happens in background subshells. A marker inside the
per-run temporary directory communicates a fatal processing failure to the
parent shell. The marker records the phase and return code.

The parent checks the marker after each `wait -n` and after the final `wait`. If
present, it stops queueing feeds, waits for already-running workers, skips merge
and final rendering, reports an error status, and runs cleanup. In particular,
a `sort` failure such as exit status 137 cannot be treated as a recoverable feed
failure followed by another download.

The preparation log reports the actual sorting/preparation return code rather
than the earlier download return code.

### 5. Transactional blocklist update

The existing active backend file is not truncated during update admission.
Rendering writes to a file in the per-run temporary directory. Only after all
feeds have been prepared, merged, rendered, counted, and validated as non-empty
is that file renamed over the active SmartDNS blocklist.

If admission fails, a worker fails, the shell receives a catchable signal, or
rendering fails, the old blocklist remains in place. First-run failure leaves the
existing empty/header-only state unchanged.

### 6. Cleanup and signals

After the per-run temporary directory is created, adblock registers cleanup for
normal exit and `HUP`, `INT`, and `TERM`. Cleanup is idempotent and only removes
the exact temporary directory created by the current process. The existing stop
path continues to remove normal runtime state.

`SIGKILL` cannot be trapped. The fail-fast parent handling addresses the
observed case where the OOM killer kills a child sorter. A subsequent invocation
also removes stale adblock temporary directories that match the package-created
template and are not owned by a live adblock process. Cleanup must validate the
base directory and generated name before recursive removal.

## Error handling

- Insufficient headroom: log an explicit low-memory error, retain active rules,
  clean temporary files, and return non-zero.
- Download failure: retain the existing per-feed restore behavior unless a
  fatal memory/processing marker exists.
- Sort or preparation failure: mark the complete update failed and stop adding
  work.
- Merge/render validation failure: do not rename the candidate file over the
  active file.
- Cleanup failure: log the exact path and error but preserve the primary update
  result.

## Testing

Add a shell regression harness under `feeds/packages/net/adblock/tests/`. A test
mode lets the script define its functions without connecting to ubus or running
an action. Tests use temporary mock meminfo files and command stubs.

Required cases:

1. `MemAvailable` without swap below 16 MiB is rejected.
2. The same RAM state with sufficient `SwapFree` is accepted.
3. Invalid or missing meminfo values are rejected.
4. The pre-sort requirement includes the configured sort buffer.
5. Successful feed preparation removes raw/intermediate input.
6. Sorting failure creates a fatal marker and prevents merge/final replacement.
7. Signal/exit cleanup removes only the current run directory.
8. Failed rendering leaves an existing active blocklist byte-for-byte intact.
9. A static target test confirms `Device/unicom_vs010` includes `zram-swap`.

Verification also includes:

- POSIX shell syntax checking with `sh -n`;
- the new regression harness;
- OpenWrt package compilation for `adblock` and `zram-swap`;
- target metadata/image package verification for `unicom_vs010`;
- hardware validation over serial: three-feed reload completes without OOM,
  `/proc/swaps` contains zram, temporary files return to baseline, and the old
  blocklist survives an induced low-memory rejection.

## Success criteria

- A freshly built VS010 image starts with active zram swap.
- The existing three-feed reload no longer causes `MemAvailable` to remain at
  zero, an OOM kill, or multi-second serial-shell reclaim stalls.
- A deliberately insufficient-memory update exits before destructive changes.
- Child sorting failure does not start the next feed.
- `/tmp` adblock working files are removed after success and failure.
- The active SmartDNS blocklist changes only after a complete successful update.
- Existing WAN, LAN, Wi-Fi, and DNS behavior remains functional.
