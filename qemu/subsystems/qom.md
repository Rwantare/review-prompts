# QEMU Object Model (QOM) & Device Architecture

## 1. QOM Lifecycle Invariants

QOM provides object-oriented infrastructure for QEMU. Every emulated device is a QOM object inheriting from `TYPE_DEVICE`.

```
Object -> DeviceState -> [SysBusDevice | PCIDevice | VirtIODevice | ...] -> MyDeviceState
```

### 1.1 Method Responsibilities

| Lifecycle Phase | Function | Allowed Operations | Forbidden Operations |
|-----------------|----------|-------------------|----------------------|
| **Class Init** | `class_init` | Set method pointers (`dc->realize`), register properties (`device_class_set_props`), assign `dc->vmsd`, set device categories. | Accessing instance data, allocating memory for device instances. |
| **Instance Init** | `instance_init` | Set default scalar values, `object_initialize_child`, `memory_region_init_io` (unmapped). | Opening FDs/sockets, allocating host buffers, mapping memory to buses, connecting IRQs, calling `exit()`, `error_fatal`. |
| **Realize** | `DeviceClass.realize` | Validate configuration, allocate buffers, map memory regions, connect IRQs, attach to backends. | Leaking resources on error return. |
| **Unrealize** | `DeviceClass.unrealize` | Disconnect IRQs, unmap memory regions, free buffers allocated in `realize`. | Leaving active timers or pending bottom halves (BH). |
| **Instance Finalize**| `instance_finalize`| Free memory allocated in `instance_init`, unparent children. | Assuming `realize` was ever called. |

---

## 2. Realize Error Unwinding Pattern

QEMU **does not invoke `unrealize`** if `realize` returns an error. The `realize` function must clean up every resource it acquired before returning failure:

```c
static void my_device_realize(DeviceState *dev, Error **errp)
{
    MyDeviceState *s = MY_DEVICE(dev);
    ERRP_GUARD();

    /* 1. Validate properties */
    if (s->queue_size > MAX_QUEUE_SIZE) {
        error_setg(errp, "queue_size %u exceeds maximum %u",
                   s->queue_size, MAX_QUEUE_SIZE);
        return;
    }

    /* 2. Allocate runtime buffers */
    s->buffer = g_try_malloc0(s->buffer_size);
    if (!s->buffer) {
        error_setg(errp, "failed to allocate device buffer");
        return;
    }

    /* 3. Initialize backend (fallible) */
    if (!init_backend(s, errp)) {
        /* CRITICAL: Must clean up s->buffer before returning! */
        g_free(s->buffer);
        s->buffer = NULL;
        return;
    }
}
```

---

## 3. QOM Cast Macros & Type Declarations

Modern QEMU defines types using declarative macros in header files:

```c
#define TYPE_MY_DEVICE "my-device"
OBJECT_DECLARE_SIMPLE_TYPE(MyDeviceState, MY_DEVICE)

struct MyDeviceState {
    /* Private parent must be first field */
    SysBusDevice parent_obj;

    /* Public/device fields */
    uint32_t reg_status;
    MemoryRegion mmio;
};
```

- **`MY_DEVICE(obj)`**: Safely casts a generic `Object *` to `MyDeviceState *`. In debug builds, this validates the QOM type hierarchy via `OBJECT_CHECK()`.
- **Anti-Pattern**: Never use raw C pointer casts (`(MyDeviceState *)obj`) on QOM objects. Always use the generated QOM cast macro.

---

## 4. Reset Protocol (`ResettableClass`)

Devices should participate in the hierarchical reset framework (`docs/devel/reset.rst`):
- Do NOT use legacy `qemu_register_reset()` for device models.
- Implement `ResettableClass` phases (`phases.enter`, `phases.hold`, `phases.exit`). Avoid deprecated `device_class_set_legacy_reset()` in new code:
  - **Phase Enter**: Reset internal registers and state latches. Do NOT trigger external IRQs or notify neighbors.
  - **Phase Hold**: Perform intermediate transitions.
  - **Phase Exit**: Update output lines (e.g. `qemu_set_irq`) to reflect post-reset state.
- **Invariant**: Reset must be repeatable without leaking memory. Resetting a device 100 times must not consume additional host memory.

---

## 5. Common QOM Bug Patterns

1. **Host Allocation in `instance_init`**:
   - Symptom: Running `qemu-system-x86_64 -device my-device,help` leaks host memory, open sockets, or creates host files.
2. **Missing `ERRP_GUARD()` or Unchecked `*errp`**:
   - Symptom: Dereferencing `*errp` (e.g. `if (*errp)`) when caller passed `NULL` causes an immediate SIGSEGV. Alternatively, if an earlier sub-call sets `*errp` and execution continues into a second fallible function, `error_setg()` asserts (`assert(*errp == NULL)`).
3. **Double Free in `unrealize`**:
   - Freeing resources that were already freed or were managed by child QOM objects.
4. **Active Timers Across Unrealize**:
   - Forgetting to call `timer_del()` / `timer_free()` in `unrealize`, causing a timer callback to fire on a freed device.
