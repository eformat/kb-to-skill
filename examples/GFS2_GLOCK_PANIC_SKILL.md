---
name: gfs2-glock-panic
description: Diagnose and remediate kernel panics caused by GFS2 glock BUG at fs/gfs2/glock.c on RHEL 8 and RHEL 9 cluster nodes. Use this skill whenever a user mentions kernel panics with gfs2, glock.c, finish_xmote, glock_work_func, lm_lock errors, or DLM lock issues in HA cluster environments — even if they just paste a stack trace containing these symbols. Also use when troubleshooting GFS2 filesystem hangs or crashes on Pacemaker/Corosync clusters. Based on https://access.redhat.com/solutions/7123129
---

## Panic Signatures

Match against these patterns in `vmcore-dmesg.txt` or system logs to confirm this is the issue.

RHEL 8 (kernel 4.18.0-553.x):
```
kernel BUG at fs/gfs2/glock.c:650!
RIP: 0010:finish_xmote.cold.70+0x35/0x37 [gfs2]
Workqueue: gfs2-glock/<fsname> glock_work_func [gfs2]
```

RHEL 9 (kernel 5.14.0-570.x):
```
kernel BUG at fs/gfs2/glock.c:646!
RIP: 0010:finish_xmote.cold+0x38/0x3a [gfs2]
Workqueue: gfs2-glock/<fsname> glock_work_func [gfs2]
```

The `lm_lock ret-22` messages typically appear just before the panic — these mean the DLM is rejecting lock requests (`-EINVAL`):
```
gfs2: fsid=<cluster>:<fsname>: lm_lock ret-22
gfs2: fsid=<cluster>:<fsname>: wanted 0 got 1
```

## Diagnose

Confirm the panic signature from crash dumps.

```bash
grep -E "kernel BUG at fs/gfs2/glock.c:(646|650|670)" /var/crash/*/vmcore-dmesg.txt
```

Identify which filesystem and cluster are affected — the `fsid=cluster:fsname` field tells you.

```bash
grep "lm_lock ret" /var/crash/*/vmcore-dmesg.txt
```

Check the running kernel version to determine which errata applies.

```bash
uname -r
```

Check GFS2 mount status on surviving nodes.

```bash
mount | grep gfs2
```

Review DLM health — lock issues between nodes often precede this panic.

```bash
dlm_tool status
dlm_tool lockdebug <fsname>
```

Check overall cluster health.

```bash
pcs status
corosync-quorumtool -s
```

## Root Cause

The GFS2 glock state machine hits an assertion failure in `finish_xmote` — a lock transition returned an unexpected state. The DLM rejects lock requests (the `lm_lock ret-22` / `-EINVAL` errors) due to inter-node communication issues, a node being fenced mid-operation, or a race condition in concurrent lock operations.

## Resolve

This is a known kernel bug. Apply the kernel update that contains the fix.

RHEL 8:
```bash
yum update kernel
```

RHEL 9:
```bash
dnf update kernel
```

Reboot cluster nodes in a rolling fashion — never reboot all nodes simultaneously or you lose quorum.

```bash
pcs node standby <nodename>
# reboot the node
pcs node unstandby <nodename>
```

## Interim Workaround

If a kernel update cannot be applied immediately, monitor for the precursor `lm_lock` errors and proactively fence the affected node before the panic cascades to other nodes.

```bash
journalctl -k -f | grep "lm_lock ret"
```

Verify kdump is configured so crash dumps are captured for Red Hat support analysis.

```bash
systemctl status kdump
kdumpctl showmem
```

## Related Errata

- glock.c:670 variant: https://access.redhat.com/solutions/7021403 — resolved in RHSA-2023:7077 (RHEL 8) and RHSA-2023:6583 (RHEL 9)
- Product: Red Hat Enterprise Linux 8, 9
- Components: kernel, cluster (GFS2, DLM)
