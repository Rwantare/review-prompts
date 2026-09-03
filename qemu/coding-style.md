# QEMU Coding Style Guide

This guide summarizes the official QEMU coding conventions documented in `docs/devel/style.rst` (formerly `CODING_STYLE.rst`). AI assistants and human reviewers must enforce these rules and run `scripts/checkpatch.pl` on all patch submissions.

---

## 1. Whitespace & Indentation

- **Indentation**: Exactly **4 spaces**. Do NOT use tabs anywhere in C files.
- **Tab characters**: Strictly forbidden. `scripts/checkpatch.pl` will flag any tab character in patch diffs.
- **Line Length**: Lines should not exceed **80 characters**. If wrapping impairs readability (e.g. long strings or complex function prototypes), lines up to 90-100 characters may be accepted, but 80 remains the standard.
- **Trailing Whitespace**: Strictly forbidden at line endings or on blank lines.

---

## 2. Braces `{}` are Mandatory

Unlike the Linux kernel, QEMU **requires braces** on all `if`, `else`, `while`, `for`, and `do` statements, even when the statement body is a single line:

```c
/* CORRECT in QEMU: */
if (condition) {
    do_something();
} else {
    do_other();
}

/* INCORRECT (violates QEMU style): */
if (condition)
    do_something();
```

**Brace Placement**:
- Control statements: Opening brace goes on the same line as the keyword (`if (...) {`).
- Function definitions: Opening brace goes on its own line at column 0:
  ```c
  static void my_function(void)
  {
      /* body */
  }
  ```

---

## 3. Types and Typedefs

- **Standard C99 Types**:
  - Unsigned: `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t`.
  - Signed: `int8_t`, `int16_t`, `int32_t`, `int64_t`.
  - Boolean: Use `bool`, `true`, `false` (`<stdbool.h>`).
  - Do NOT use Linux kernel types (`u8`, `u16`, `u32`, `u64`, `s32`) in native QEMU code. (Exception: Linux headers under `include/standard-headers/` imported directly from the kernel).
- **Structure Typedefs**:
  - Structures should be typedef'd with `CamelCase` names:
    ```c
    typedef struct MyDeviceState MyDeviceState;
    struct MyDeviceState {
        DeviceState parent_obj;
        /* fields */
    };
    ```
- **Pointers**:
  - The asterisk binds to the variable name, not the type:
    ```c
    char *name;   /* CORRECT */
    char* name;   /* INCORRECT */
    ```

---

## 4. Naming Conventions

- **Variables and Functions**: `lower_case_with_underscores` (`snake_case`).
- **Types, Structs, Classes**: `CamelCase` (`MyDeviceState`, `VirtIODevice`).
- **Constants, Macros, Enums**: `UPPER_CASE_WITH_UNDERSCORES` (`TYPE_PCI_DEVICE`, `MY_REG_OFFSET`).
- **Subsystem Prefixes**: Device models and files should consistently prefix their functions with the device/driver name (e.g., `nvme_*`, `virtio_net_*`).

---

## 5. Comments

- **Block Comments**:
  ```c
  /*
   * Header line explaining context.
   * Subsequent detailed explanation.
   */
  ```
- **Single-Line Comments**:
  ```c
  /* Short comment */
  ```
- Avoid C++ `//` comments in core C code; prefer classic C comments `/* ... */`.

---

## 6. Error & Diagnostic Messages

- **Reporting Functions**:
  - `error_report("error message here")`
  - `warn_report("warning message here")`
  - `info_report("status message here")`
- **Formatting Invariants**:
  - Do NOT include a trailing newline (`\n`). The reporting function appends it.
  - Do NOT include a trailing period (`.`).
  - Do NOT prefix with "Error:" or "Warning:" (the reporting helper adds appropriate prefixes).
- **Guest Logging**:
  - Use `qemu_log_mask(LOG_GUEST_ERROR, "%s: ...\n", __func__, ...)` for invalid guest register accesses or corrupted descriptor rings. Note: `qemu_log_mask` DOES require a trailing newline `\n`.

---

## 7. Mechanical Verification via `checkpatch.pl`

Before submitting or approving a patch:
```bash
./scripts/checkpatch.pl <patch-file>
```
Or for git commits:
```bash
./scripts/checkpatch.pl --branch master..HEAD
```

Address all reported errors and warnings unless an explicit exception applies (e.g. importing external code from imported Linux headers).
