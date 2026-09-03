# QEMU Review Report Template

## Instructions

When bugs, regressions, or critical violations are verified, generate `review-inline.txt` following this template.

## Formatting Rules

1. **Plain text only**: No Markdown formatting (no `**bold**`, no backticks for code).
2. **Wrap at 78 characters**: Wrap text paragraphs at 78 characters to match mailing list standards. Code snippets can remain unwrapped.
3. **Identify by function and context**: Do not use raw patch line numbers alone; refer to file paths, function names, and surrounding code blocks (line numbers change between patch versions).
4. **Factual and constructive tone**: State what happens, why it is problematic, and how to fix it according to QEMU invariants.
5. **Ready for `qemu-devel@nongnu.org`**: The output should be directly pastable as a mailing list reply or GitLab MR review comment.

---

## Template

```text
Subject: Re: [PATCH] <original subject line>

Thanks for the patch. During review, I identified potential issues that
should be addressed:

=== Issue 1: <Brief summary of the issue> ===

File: <path/to/file.c>
Function: <function_name>()
Severity: <CRITICAL | HIGH | MEDIUM | LOW>

Description:
<Explain the issue clearly in 78-character wrapped prose. Explain what goes
wrong, under what conditions, and what invariant is broken.>

Problematic code:

    <code snippet showing the issue>
    <use 4-space indentation, no markdown backticks>

Why this is an issue:
<Detailed technical rationale, e.g. guest denial of service, memory leak on
error unwind, missing ERRP_GUARD, or migration stream desynchronization.>

Trigger condition:
<Explain how this path is reached: by guest MMIO write, QMP command, or error path.>

Suggested fix:

    <replacement code snippet fixing the problem>

---

=== Issue 2: <Brief summary of second issue> ===

[Repeat format for each verified issue]

---

Analysis notes:
- <Call paths and functions traced>
- <Verification against false-positive-guide.md>
- <Subsystems checked>
```

---

## Concrete QEMU Example

```text
Subject: Re: [PATCH v2 3/5] hw/net/my_nic: Add support for jumbo frames

Thanks for the patch. During review, I identified potential issues that
should be addressed:

=== Issue 1: Guest-triggerable assertion failure via invalid MTU ===

File: hw/net/my_nic.c
Function: my_nic_set_mtu()
Severity: CRITICAL

Description:
The device model uses assert() to validate the MTU value written by the
guest to the NIC_REG_MTU register. Because guest code controls this
register value directly via MMIO, a malicious or buggy guest OS kernel can
cause an assertion failure in the host QEMU process, causing a denial of
service of the virtual machine.

Problematic code:

    static void my_nic_write(void *opaque, hwaddr addr, uint64_t val,
                             unsigned size)
    {
        MyNicState *s = opaque;

        switch (addr) {
        case NIC_REG_MTU:
            assert(val <= MY_NIC_MAX_MTU);
            s->mtu = val;
            break;

Why this is an issue:
QEMU device models must never crash or assert due to guest-supplied
values. Guest input is untrusted. Invalid operations must be reported
using qemu_log_mask(LOG_GUEST_ERROR, ...) and clamped or ignored.

Trigger condition:
Guest OS writes any value greater than MY_NIC_MAX_MTU (e.g. 0xffff) to the
NIC_REG_MTU register.

Suggested fix:

        case NIC_REG_MTU:
            if (val > MY_NIC_MAX_MTU) {
                qemu_log_mask(LOG_GUEST_ERROR,
                              "%s: MTU %" PRIu64 " exceeds limit %u\n",
                              __func__, val, MY_NIC_MAX_MTU);
                break;
            }
            s->mtu = val;
            break;

---

=== Issue 2: Resource leak on realize failure ===

File: hw/net/my_nic.c
Function: my_nic_realize()
Severity: HIGH

Description:
If the call to my_nic_init_backend() fails during device realization, the
function returns without freeing the buffer allocated earlier by
g_malloc0() for s->rx_buf. QEMU does not call unrealize if realize encounters
an error, causing s->rx_buf to leak permanently.

Problematic code:

    static void my_nic_realize(DeviceState *dev, Error **errp)
    {
        MyNicState *s = MY_NIC(dev);

        s->rx_buf = g_malloc0(s->mtu + MY_NIC_HEADER_LEN);

        if (!my_nic_init_backend(s, errp)) {
            return;
        }
    }

Suggested fix:

    static void my_nic_realize(DeviceState *dev, Error **errp)
    {
        MyNicState *s = MY_NIC(dev);

        s->rx_buf = g_malloc0(s->mtu + MY_NIC_HEADER_LEN);

        if (!my_nic_init_backend(s, errp)) {
            g_free(s->rx_buf);
            s->rx_buf = NULL;
            return;
        }
    }

---

Analysis notes:
- Traced MMIO dispatch path from memory.c to my_nic_write()
- Confirmed BQL is held during MMIO dispatch
- Verified s->rx_buf is not automatically freed on realize error exit
```
