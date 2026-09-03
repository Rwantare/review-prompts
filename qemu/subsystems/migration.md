# QEMU Live Migration & VMState Subsystem Guide

Live migration allows a running guest virtual machine to transfer seamlessly between two QEMU processes (potentially across different physical hosts and QEMU versions) without interrupting the guest.

---

## 1. The `VMStateDescription` Structure

Device state is serialized using declarative `VMStateDescription` structures (`include/migration/vmstate.h`):

```c
static const VMStateDescription vmstate_my_device = {
    .name = "my-device",
    .version_id = 2,
    .minimum_version_id = 1,
    .pre_load = my_device_pre_load,
    .post_load = my_device_post_load,
    .fields = (const VMStateField[]) {
        VMSTATE_UINT32(status, MyDeviceState),
        VMSTATE_UINT32(control, MyDeviceState),
        VMSTATE_UINT32(ring_head, MyDeviceState),
        VMSTATE_BUFFER(scratchpad, MyDeviceState),
        VMSTATE_END_OF_LIST()
    },
    .subsections = (const VMStateDescription * const []) {
        &vmstate_my_device_extended_feature,
        NULL
    }
};
```

---

## 2. Stream Compatibility Invariants

QEMU guarantees forward and backward migration between adjacent releases for versioned machine types (e.g. `pc-q35-8.2` or `virt-8.2`).

### 2.1 The Golden Rules of VMState Fields

1. **Never Reorder Fields**: The migration stream is a continuous byte stream. Reordering fields breaks deserialization.
2. **Never Delete Fields**: Removing an existing field desynchronizes the stream.
3. **Never Change Types or Sizes**: Changing `VMSTATE_UINT32` to `VMSTATE_UINT64` corrupts all subsequent fields.
4. **Never Add Unconditional Fields**: Adding a new `VMSTATE_*` field to `.fields` breaks migration from older QEMU versions that do not transmit that field.

---

## 3. Subsections for Extensible State

To add new state without breaking backwards migration, use a **VMState subsection**:

```c
static bool extended_feature_needed(void *opaque)
{
    MyDeviceState *s = opaque;
    /* Only send this subsection if the feature is non-default/active */
    return s->extended_feature_enabled;
}

static const VMStateDescription vmstate_my_device_extended_feature = {
    .name = "my-device/extended_feature",
    .version_id = 1,
    .minimum_version_id = 1,
    .needed = extended_feature_needed,
    .fields = (const VMStateField[]) {
        VMSTATE_UINT32(extra_reg, MyDeviceState),
        VMSTATE_END_OF_LIST()
    }
};
```

- **How It Works**:
  - If `needed()` returns `false`, QEMU omits the subsection entirely, allowing migration to/from older QEMU releases.
  - If `needed()` returns `true`, the subsection is tagged with its name in the stream.

---

## 4. Machine Type Compatibility (`hw_compat_*`)

If a newly added field or feature is enabled by default:
- It **must** be disabled for older machine types to preserve backwards migration.
- Add an entry in `hw_compat_*` (in `hw/core/machine.c`):
  ```c
  GlobalProperty hw_compat_8_2[] = {
      { "my-device", "extended-feature", "off" },
  };
  ```

---

## 5. Security & Post-Load Validation (`post_load`)

The migration stream must be treated as **untrusted input**. A corrupted stream or hostile sender must not be able to compromise the receiving host.

### 5.1 Mandatory Validations in `post_load`

Always implement `.post_load` to validate state variables loaded from the stream:

```c
static int my_device_post_load(void *opaque, int version_id)
{
    MyDeviceState *s = opaque;

    /* 1. Validate ring buffer head/tail indices */
    if (s->ring_head >= MY_RING_SIZE || s->ring_tail >= MY_RING_SIZE) {
        error_report("%s: invalid ring indices: head=%u tail=%u",
                     __func__, s->ring_head, s->ring_tail);
        return -EINVAL; /* abort migration cleanly */
    }

    /* 2. Validate payload lengths */
    if (s->packet_len > sizeof(s->packet_buf)) {
        error_report("%s: packet length %u exceeds buffer",
                     __func__, s->packet_len);
        return -EINVAL;
    }

    return 0;
}
```

- **Modern `errp` Callbacks (`post_load_errp`, `pre_load_errp`, `pre_save_errp`)**:
  - Upstream QEMU (`docs/devel/migration/main.rst`) encourages replacing legacy `int (*)(void *, int)` callbacks with `errp` variants:
    - `bool (*pre_load_errp)(void *opaque, Error **errp);`
    - `bool (*post_load_errp)(void *opaque, int version_id, Error **errp);`
    - `bool (*pre_save_errp)(void *opaque, Error **errp);`
  - New implementations should preferentially use these `_errp` methods so error descriptions propagate cleanly through QMP and migration channels instead of returning raw negative errno values.
- **Security Rule**: If `post_load` or `post_load_errp` fails to bounds-check an index or length loaded from the migration stream that is subsequently used to access an array or buffer, flag it as a **CRITICAL** vulnerability (potential host arbitrary code execution).
