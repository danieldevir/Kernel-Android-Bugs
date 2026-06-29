# 🦁 Daniel Baradaran – Vulnerability Research & Bug Bounty Portfolio

> *11 years old | Self‑taught security researcher | Kernel & OS internals*

---

## 📌 About Me

I’m an 11‑year‑old security researcher from Iran.  
I started programming at 8, and today I focus on **static analysis** of the Android, iOS (XNU), and Qualcomm kernels.

- 🧠 **IQ:** 137  
- 💻 **Tools:** `grep`, `Coccinelle`, `nano`, Termux, `adb`  
- ⚙️ **Hardware:** 15‑year‑old laptop (2 GB RAM) + 12‑year‑old phone (Sony Xperia Z2)  

All my work is done with minimal resources – proving that you don’t need expensive gear to find serious vulnerabilities.

---

## 🔥 Key Achievements

| Vendor | Status | Priority / Severity |
|--------|--------|---------------------|
| **Google (Android)** | ✅ Assigned, waiting for CVE | P2 / S2 |
| **Apple (XNU)** | ⏳ Under review | – |
| **Qualcomm** | ✅ Acknowledged (QPSIIR‑2072) | – |

> I have submitted **multiple valid vulnerabilities** to Google, Apple, and Qualcomm.  
> This page lists them with technical details, PoC code, and fixes.

---

## 🎯 1. Google Android – Negative Array Index in `vchiq_debugfs.c`

**Status:** ✅ **Assigned & waiting for CVE**  
**Priority:** P2 | **Severity:** S2  
**Issue Tracker:** [#525341492](https://issuetracker.google.com/issues/525341492)

### Vulnerability
A negative array index bug exists in the debugfs write handler.

**File:** `drivers/staging/vc04_services/interface/vchiq_arm/vchiq_debugfs.c`

**Vulnerable Code:**
```c
char kbuf[DEBUGFS_WRITE_BUF_SIZE + 1];
...
if (copy_from_user(kbuf, buffer, count))
    return -EFAULT;
kbuf[count - 1] = 0;   // ❌ if count == 0 → kbuf[-1]
```

If a user writes `count = 0` to `/sys/kernel/debug/vchiq/log_level`, the kernel accesses `kbuf[-1]`, causing an out‑of‑bounds write → kernel panic.

**Exploit (PoC):**
```c
int fd = open("/sys/kernel/debug/vchiq/log_level", O_WRONLY);
write(fd, "", 0);   // count = 0
close(fd);
```

**Proposed Fix:**
```c
if (count == 0)
    return -EINVAL;
kbuf[count - 1] = 0;
```

---

## 📱 2. Apple (XNU) – Buffer Overflow in `IOSharedDataQueue::enqueue()`

**Status:** ⏳ Closed (reopen with PoC)  
**Report ID:** OE110646826475

### Vulnerability
`IOSharedDataQueue::enqueue()` uses `_nochkmemcpy` without validating the `dataSize` from userspace.

**Vulnerable Code:**
```cpp
void IOSharedDataQueue::enqueue(void *data, uint32_t dataSize) {
    ...
    _nochkmemcpy(dest, data, dataSize);   // ❌ no bounds check
}
```

If `dataSize` is `0xFFFFFFFF` or `-1`, the copy operation writes past the allocated buffer.

**Expected Impact:** Kernel panic or memory corruption.

**Apple’s Response:**  
They require a **crash log** from a current build.  
Since I don’t have a Mac/iOS device, the report is closed – but the bug exists.

---

## 🧠 3. Apple (XNU) – TOCTOU / Double Fetch in `ipc_kmsg.c`

**Status:** ⏳ Under review  
**Report ID:** OE11064689512918

### Vulnerability
The kernel fetches the same userspace value twice, allowing a race condition.

**Affected File:** `osfmk/ipc/ipc_kmsg.c`

**Flow:**
1. First fetch (line 2980): reads `descriptor_count`
2. Malicious thread modifies the value
3. Second fetch (line 3018): uses the changed value

This is the same pattern as **CVE-2021-30955** (TOCTOU in `mach_msg`).

**Impact:** Kernel memory corruption / code execution.

---

## 🎧 4. Qualcomm – Deadlock in `q6apm.c` (Android 14 & 15)

**Status:** ✅ **Acknowledged** (QPSIIR‑2072)  
**Ticket:** QPSIIR‑2072

### Vulnerability
`q6apm_graph_alloc()` locks `apm->lock` **after** the error path, so if `audioreach_alloc_graph_pkt()` fails, the function returns without ever releasing the lock – causing a permanent deadlock.

**File:** `sound/soc/qcom/qdsp6/q6apm.c`

**Vulnerable Code:**
```c
graph->graph = audioreach_alloc_graph_pkt(apm, info);
if (IS_ERR(graph->graph)) {
    kfree(graph);
    return ERR_CAST(err);          // ❌ lock never acquired → logical deadlock
}
mutex_lock(&apm->lock);            // Only reached if no error
```

**Fix:**
Move the lock **before** the allocation and error path.

---

## 🧪 5. Google Android – Multiple Bugs in `atomisp` (Staging Driver)

**Status:** ✅ Submitted (awaiting review)  
**Type:** Integer overflow + missing error checks

### Findings in `drivers/staging/media/atomisp/pci/sh_css_params.c`

| Bug | Description |
|-----|-------------|
| **Integer overflow** | `height * width` overflows to 0 before `kvmalloc()` |
| **Missing NULL check** | `coordinates_x[i]` / `coordinates_y[i]` are not checked after allocation |
| **NULL return** | Function returns `NULL` instead of `-ENOMEM` on error |

---

## 🛠️ Methodology

I use **static source‑code analysis** with:

```bash
grep -r --include="*.c" -B 5 -A 10 "copy_from_user" drivers/
coccinelle --sp-file kmalloc_check.cocci --dir . --no-includes
```

My workflow:
1. Download latest kernel (`android-mainline`, `XNU`, `Qualcomm` MSM)
2. Search for dangerous patterns (`copy_from_user`, `kmalloc` without NULL checks, `mutex_lock` / `unlock` imbalances)
3. Manually verify every finding with `nano` and full context
4. Write a report with PoC and fix, then submit via official VRP channels

---

## 🏆 Summary

| Company | Bug Type | Status |
|---------|----------|--------|
| Google Android | Negative array index | ✅ Assigned – CVE in progress |
| Apple XNU | Buffer overflow | ⏳ Closed – needs PoC |
| Apple XNU | TOCTOU | ⏳ Under review |
| Qualcomm | Deadlock | ✅ Acknowledged |
| Google Android (atomisp) | Integer overflow + missing checks | ✅ Submitted |

---

## 🧠 Why I Do This

I believe that **security is a right, not a privilege**.  
I have no fancy hardware, no expensive tools – just a curious mind and a lot of persistence.

If I can find bugs with a 2‑GB RAM laptop, so can you.

**Stay curious. Stay kind.**

— **Daniel Baradaran**  
🔗 [GitHub](https://github.com/danieldevir) · 
📧 daniel.ir.dev@gmail.com
