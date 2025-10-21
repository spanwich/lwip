# seL4 Research Team lwIP Customizations

This fork contains safety improvements for lwIP in seL4 environments.

## Branch: seL4-safe

Based on: `STABLE-2_1_2_RELEASE` (commit 159e31b689577dbf69cf0683bbaffbd71fa5ee10)

## Modifications

### 1. seL4-SAFE: Prevent use-after-free in tcp_abandon()

**File:** `src/core/tcp.c`
**Lines:** 604-637
**Branch:** seL4-safe
**Date:** 2025-10-21

**Change:** NULL segment list pointers immediately after freeing

**Code Modified:**
```c
// Added after tcp_segs_free() calls:
pcb->unacked = NULL;  /* seL4-SAFE: Break dangling pointer */
pcb->unsent = NULL;   /* seL4-SAFE: Break dangling pointer */
pcb->ooseq = NULL;    /* seL4-SAFE: Break dangling pointer */
```

**Root Cause:**
`tcp_abandon()` frees PCB segment lists (`unacked`, `unsent`, `ooseq`) but error callback fires afterward (line 622). If callback triggers code that accesses these pointers, use-after-free occurs causing crashes at offset 0x10 when dereferencing freed segment structures.

**Why This Matters in seL4:**
seL4's strong memory protection turns silent memory corruption into immediate, debuggable crashes. The use-after-free bug that would be difficult to detect in other systems manifests as consistent page faults at `tcp_output:1389` when accessing `useg->tcphdr->seqno` where `useg` points to freed memory.

**Fix Impact:**
- **Zero functional changes** - lwIP behavior unchanged
- **Prevents race condition crashes** - Callbacks see NULL instead of freed memory
- **Leverages existing NULL checks** - Functions like `tcp_output()` already have NULL guards (line 1381)
- **seL4-appropriate** - Deterministic, no timing dependencies

**Testing:**
- Platform: seL4 microkernel on ARM qemu-arm-virt
- Scenario: ICS gateway with rapid connection cleanup (SCADA connect/disconnect cycles)
- Result: No crashes after fix (previously crashed consistently at tcp_output:1389)
- Evidence: Page fault address 0x10 (NULL+16 bytes, accessing freed struct field)

**Upstream Status:**
Under consideration for submission to lwIP project as general safety improvement.

---

## Merging Upstream Updates

When merging new lwIP versions:

1. **Check tcp_abandon() changes:**
   ```bash
   git diff STABLE-2_1_2_RELEASE..upstream/master -- src/core/tcp.c
   ```

2. **Ensure NULL assignments remain** after `tcp_segs_free()` calls

3. **Re-test with stress test:**
   - 100+ rapid connection create/destroy cycles
   - Monitor for page faults at tcp_output:1389
   - Check GDB fault registers (should all be 0x0)

4. **Verify no regressions:**
   - seL4 component stability (both Net0 and Net1 running)
   - Connection tracking accuracy (create count == close count)
   - No lwIP assertions

---

## Repository Information

**Upstream:** https://git.savannah.gnu.org/git/lwip.git
**Fork:** https://github.com/spanwich/lwip.git
**Branch:** seL4-safe
**Base Tag:** STABLE-2_1_2_RELEASE

**Maintainer:** seL4 ICS Gateway Research Team
**Contact:** spanwich (GitHub)

---

## Why Fork Instead of Patch?

**Advantages of Fork:**
1. ✅ Fixes root cause at source (not workaround in application)
2. ✅ Benefits all code paths (not just specific cleanup scenarios)
3. ✅ Minimal maintenance (3-line change, well-documented)
4. ✅ Future-proof (prevents similar bugs in new code)
5. ✅ Upstream potential (could be merged back to lwIP)

**Alternative Considered:**
Defensive checks in driver code - rejected because:
- Treats symptoms, not root cause
- Requires checks scattered throughout codebase
- Easy to miss in future development
- Higher maintenance burden

---

## Development Notes

**seL4 Memory Protection as Debug Tool:**

This bug demonstrates seL4's value for safety-critical development:
- Traditional systems: Use-after-free causes silent corruption, intermittent bugs
- seL4 system: Immediate page fault with precise fault address
- Result: Bug detection is deterministic and debuggable

**Lesson Learned:**
When seL4 crashes with fault address 0x10-0x20, suspect use-after-free accessing struct fields in freed memory. The offset corresponds to field position in the freed structure.

---

## Version History

**v1.0 (2025-10-21):**
- Initial seL4-safe branch created
- Applied tcp_abandon NULL pointer fix
- Documented customizations
- Tested on ICS gateway application

**Next Steps:**
- Monitor lwIP upstream for similar fixes
- Consider submitting patch upstream
- Evaluate merging newer lwIP releases
