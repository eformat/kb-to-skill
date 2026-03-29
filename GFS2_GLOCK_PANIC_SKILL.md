---
name: gfs2-glock-panic
description: Diagnose and remediate kernel panics caused by GFS2 glock BUG at fs/gfs2/glock.c (lines 646, 650, 670) on RHEL 8 and RHEL 9 cluster nodes. Use this skill when investigating kernel panics involving gfs2, glock, finish_xmote, glock_work_func, or lm_lock errors in HA cluster environments. Based on https://access.redhat.com/solutions/7123129
---

## Identifying the Issue

A cluster node running a GFS2 filesystem panics with a BUG in `fs/gfs2/glock.c`. The panic originates in the glock subsystem during lock state transitions.

Key signatures in `vmcore-dmesg.txt` or system logs:

**RHEL 8** (kernel 4.18.0-553.x):
```
kernel BUG at fs/gfs2/glock.c:650!
invalid opcode: 0000 [#1] SMP NOPTI
RIP: 0010:finish_xmote.cold.70+0x35/0x37 [gfs2]
Workqueue: gfs2-glock/<fsname> glock_work_func [gfs2]
```

**RHEL 9** (kernel 5.14.0-570.x):
```
kernel BUG at fs/gfs2/glock.c:646!
invalid opcode: 0000 [#1] PREEMPT SMP PTI
RIP: 0010:finish_xmote.cold+0x38/0x3a [gfs2]
Workqueue: gfs2-glock/<fsname> glock_work_func [gfs2]
```

Preceding the panic you will typically see `lm_lock ret-22` errors:
```
gfs2: fsid=<cluster>:<fsname>: lm_lock ret-22
gfs2: fsid=<cluster>:<fsname>: wanted 0 got 1
```

The call trace always passes through:
1. `finish_xmote` (gfs2 glock state machine)
2. `glock_work_func` (gfs2 glock workqueue)
3. `process_one_work` (kernel workqueue)

## Diagnostic Steps

### 1. Confirm the panic signature

Check `vmcore-dmesg.txt` or `journalctl` for the BUG line:

```bash
grep -E "kernel BUG at fs/gfs2/glock.c:(646|650|670)" /var/crash/*/vmcore-dmesg.txt
```

### 2. Identify the affected filesystem and cluster

Extract the filesystem ID from the `lm_lock` messages:

```bash
grep "lm_lock ret" /var/crash/*/vmcore-dmesg.txt
```

The `fsid=<cluster>:<fsname>` field identifies the cluster and GFS2 filesystem involved.

### 3. Check the kernel version

```bash
uname -r
```

Or from the crash dump:
```bash
grep "Not tainted\|Tainted" /var/crash/*/vmcore-dmesg.txt
```

### 4. Check GFS2 mount status on surviving nodes

```bash
mount | grep gfs2
cat /proc/mounts | grep gfs2
```

### 5. Review DLM (Distributed Lock Manager) health

```bash
dlm_tool status
dlm_tool lockdebug <fsname>
```

### 6. Check cluster status

```bash
pcs status
corosync-quorumtool -s
```

## Root Cause

The panic occurs in the GFS2 glock (group lock) state machine when a lock demotion or promotion request returns an unexpected state. The `finish_xmote` function encounters a glock in a state that violates its internal assertions — specifically, the lock was expected to transition to one state but arrived in another.

The `lm_lock ret-22` messages (`-EINVAL`) indicate the DLM is rejecting lock requests, which can happen due to:

- A DLM communication issue between cluster nodes
- A node being fenced or evicted while lock operations are in flight
- A race condition in the glock state machine during concurrent lock operations
- Corruption or inconsistency in DLM lock resource state

## Resolution

### Apply the relevant kernel errata

This is a known kernel bug. Update to a kernel version that contains the fix:

**RHEL 8:**
```bash
yum update kernel
```
Check for applicable errata at https://access.redhat.com/errata/ for RHEL 8 kernel updates addressing GFS2 glock issues.

**RHEL 9:**
```bash
dnf update kernel
```
Check for applicable errata at https://access.redhat.com/errata/ for RHEL 9 kernel updates addressing GFS2 glock issues.

After updating, reboot each cluster node in a rolling fashion:

```bash
pcs node standby <nodename>
# reboot the node
pcs node unstandby <nodename>
```

### Interim workaround

If an immediate kernel update is not possible, monitor for `lm_lock ret-22` messages and proactively fence the affected node before the panic cascades:

```bash
journalctl -k -f | grep "lm_lock ret"
```

Ensure kdump is properly configured to capture vmcore for analysis:

```bash
systemctl status kdump
kdumpctl showmem
```

## Related Information

- Related solution for glock.c:670 variant: https://access.redhat.com/solutions/7021403 (resolved in RHSA-2023:7077 for RHEL 8 and RHSA-2023:6583 for RHEL 9)
- Product: Red Hat Enterprise Linux 8, 9
- Components: kernel, cluster (GFS2, DLM)
- Tags: cluster, HA, high availability, gfs2, panic, glock
