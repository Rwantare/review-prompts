# QEMU TCG & Accelerators Subsystem Guide

QEMU supports multiple execution accelerators: TCG (Tiny Code Generator, dynamic binary translation) and hardware virtualization (KVM, HVF).

---

## 1. Core Structures & Threading

- **`CPUState`**: Architecture-independent vCPU representation.
- **`CPUArchState` (`env`)**: Architecture-specific registers (x86 `CPUX86State`, ARM `CPUARMState`, RISCV `CPURISCVState`).
- **vCPU Execution**:
  - In KVM: Each vCPU runs in its own host thread executing `kvm_cpu_exec()`, switching in and out of guest mode via `ioctl(vcpu_fd, KVM_RUN)`.
  - In TCG: Multi-threaded TCG (MTTCG) allocates one host thread per vCPU.

---

## 2. TCG Translation & Helper Functions

TCG translates guest machine instructions into architecture-neutral TCG operations, which are then compiled into host machine code inside `TranslationBlock` (TB) objects.

### 2.1 TCG Helpers (`DEF_HELPER_*`)

Complex instructions that cannot be expressed easily in TCG ops call C helper functions:

```c
/* Definition in helper.h: */
DEF_HELPER_FLAGS_2(my_helper, TCG_CALL_NO_WG, void, env, i32)
```

### 2.2 Helper Flags Invariants

Helper flags inform the TCG optimizer about side effects. **Wrong flags cause silent register corruption**:

| Flag | Meaning | Required When |
|------|---------|---------------|
| `0` (default) | Reads and writes any global state, may raise exceptions. | Modifies arbitrary CPU registers or may throw guest faults. |
| `TCG_CALL_NO_WG` | Does not write global TCG variables (may read). | Reads CPU state but does not modify registers. |
| `TCG_CALL_NO_RWG` | Does not read or write global TCG variables. | Pure mathematical function with no CPU state access. |

- **CRITICAL**: If a helper function can trigger a guest exception (e.g. page fault, undefined instruction), it **CANNOT** use `TCG_CALL_NO_WG` or `TCG_CALL_NO_RWG`. The exception handler must restore exact CPU state!

---

## 3. Exception State Restoration (`cpu_restore_state`)

When an instruction causes a synchronous guest exception (e.g. MMU fault) in translated code:
- TCG uses an unwind table to reconstruct the exact guest Program Counter (PC) at the faulting instruction.
- **Invariant**: Device or CPU helpers that raise exceptions must ensure the PC is updated or call `cpu_restore_state(cpu, retaddr)` using the host return address.

---

## 4. KVM Specific Invariants

- **Capability Checks**: Never use KVM ioctls without querying support first using `kvm_check_extension(s, KVM_CAP_*)`.
- **Signal Handling**: vCPU threads handle POSIX signals to exit guest mode. Long operations in vCPU threads must not block signals.
- **Dirty Logging**: Writes to guest RAM while migration is active must be tracked via KVM dirty rings or dirty bitmaps.
