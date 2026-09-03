# QEMU QAPI & QMP Subsystem Guide

The QEMU QAPI framework (`qapi/`) defines the machine-readable schema for the QEMU Monitor Protocol (QMP), command-line options, visitor serialization, and QOM introspection.

---

## 1. Schema Definitions (`qapi/*.json`)

QAPI schemas are defined in JSON-like syntax:

```json
{ 'struct': 'MyDeviceInfo',
  'data': { 'name': 'str',
            'enabled': 'bool',
            '*queue-size': 'uint32' } }

{ 'command': 'query-my-device',
  'data': { 'name': 'str' },
  'returns': 'MyDeviceInfo' }

{ 'event': 'MY_DEVICE_ALERT',
  'data': { 'device': 'str', 'code': 'int' } }
```

- An asterisk prefix (`*queue-size`) marks a field as optional.

---

## 2. Implementing QMP Commands

Generated code defines prototypes in `qapi/qapi-commands-*.h`:

```c
MyDeviceInfo *qmp_query_my_device(const char *name, Error **errp)
{
    MyDeviceState *s = find_device(name);
    if (!s) {
        error_setg(errp, "Device '%s' not found", name);
        return NULL;
    }

    MyDeviceInfo *info = g_new0(MyDeviceInfo, 1);
    info->name = g_strdup(s->name);
    info->enabled = s->enabled;
    if (s->has_queue) {
        info->has_queue_size = true;
        info->queue_size = s->queue_size;
    }

    return info;
}
```

### 2.1 Memory Management of QAPI Types

- Generated structs are freed with `qapi_free_<TypeName>(ptr)`.
- Use `g_autoptr`: The QAPI code generator automatically emits `G_DEFINE_AUTOPTR_CLEANUP_FUNC(TypeName, qapi_free_TypeName)` in the generated headers. Do NOT declare this manually in device or command code. Simply use:
  ```c
  g_autoptr(MyDeviceInfo) info = NULL;
  ```
- **Invariant**: When returning a QAPI struct from a QMP command on success, ownership transfers to the QMP dispatcher. On error, the command must free any intermediate allocated structures (or rely on `g_autoptr` / `g_steal_pointer`).

---

## 3. Backward Compatibility & Deprecation Policy

QMP is a stable programmatic interface used by libvirt and cloud orchestrators.

1. **Strict Stability**:
   - Never rename or remove existing commands or parameters in stable releases.
   - Never change an optional parameter to mandatory.
   - Never alter the type of an existing parameter incompatibly.
2. **Deprecation Cycle**:
   - To remove or replace a command or argument:
     1. Add the `deprecated` feature flag in the QAPI schema:
        ```json
        { 'command': 'old-command',
          'features': [ 'deprecated' ] }
        ```
     2. Document the deprecation and successor in `docs/about/deprecated.rst`.
     3. Maintain the deprecated command for at least **two full QEMU releases** before deletion.

---

## 4. Documentation Comments

QAPI schema files require structured doc comments:
- Every argument and return field must be documented with `@arg: description`.
- Failure to document all arguments causes build failures during schema verification.
