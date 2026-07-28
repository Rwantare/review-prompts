# Selftests Subsystem Details

## Build System and Installation

When a new file is created in a selftests directory but not added to the
Makefile, tests fail with "No such file or directory" when run from an
installed location (via `make install`). Tests may appear to work when run
directly from the source tree because the file exists there.

The selftests build system uses several variables in each subsystem's Makefile
to control what gets installed:

| Variable | Purpose |
|----------|---------|
| `TEST_PROGS` | Executable test scripts that are run directly |
| `TEST_FILES` | Supporting files (libraries, data files, sourced scripts) |
| `TEST_GEN_FILES` | Generated binaries/files produced during build |
| `TEST_GEN_PROGS` | Generated executable test programs |

Key invariants:

- Any file referenced via `source <filename>` (bash) or `. <filename>` in
  test scripts must be added to `TEST_FILES`
- Any file referenced via `import <module>` (Python) in test scripts must be
  added to `TEST_FILES`
- Executable test scripts that are invoked directly go in `TEST_PROGS`
- Helper executables that are built during `make` go in `TEST_GEN_PROGS` or
  `TEST_GEN_FILES`

Common mistake: creating a new shared library or utility file (like
`_common.sh`, `utils.py`, `lib.sh`) that is sourced by test scripts but
forgetting to add it to `TEST_FILES`. The tests work in the source directory
but fail after `make install`.

## Result Reporting: Use the `kselftest.h` / `kselftest_harness.h` Wrappers

Hand-rolled `printf()`-based pass/fail output cannot be reliably parsed as TAP
by kselftest's own runners and by CI systems that grep for `ok`/`not ok`
lines, so failures get silently miscounted or missed by automation even
when a human reading the raw output would see them.

- Use `ksft_print_header()` and `ksft_set_plan(n)` at the start of a test
  binary, and `ksft_finished()` at the end to print the summary line — don't
  hand-format the plan/summary output.
- Report each test case via `ksft_test_result_pass()`, `ksft_test_result_fail()`,
  `ksft_test_result_skip()`, `ksft_test_result_xfail()`, or
  `ksft_test_result_xpass()` (all in `tools/testing/selftests/kselftest.h`),
  or the `ksft_test_result(condition, fmt, ...)` macro for a plain boolean.
  Use `ksft_test_result_error()` specifically for setup/environment failures
  that are distinct from the behavior under test failing.
- Exit via `ksft_exit_pass()`, `ksft_exit_fail()`, or `ksft_exit_skip(fmt, ...)`
  rather than calling `exit()` directly — these also flush the TAP summary
  first.
- For anything beyond a flat `main()` with sequential checks, use
  `TEST()`/`TEST_F()` plus `FIXTURE()`/`FIXTURE_SETUP()`/`FIXTURE_TEARDOWN()`
  and the `ASSERT_*`/`EXPECT_*` operators (`tools/testing/selftests/kselftest_harness.h`)
  instead of writing bespoke `if (...) { report failure }` blocks — the
  operators print the actual vs. expected values on failure automatically.
  `FIXTURE_VARIANT`/`FIXTURE_VARIANT_ADD` run the same test body across a
  parameter matrix instead of copy-pasting the test function per variant.

```c
// WRONG: not TAP, and the runner/CI can't tell this apart from stray stdout
if (ret != 0) {
	printf("FAIL: frobnicate returned %d\n", ret);
	return 1;
}

// CORRECT: parseable, counted, and consistent with every other test
ksft_test_result(ret == 0, "frobnicate\n");
```

**REPORT as bugs**: a test binary that prints its own ad hoc pass/fail text
instead of calling into `kselftest.h`/`kselftest_harness.h`, or that calls
`exit()`/`return` directly from `main()` without going through
`ksft_exit_*()`.

## Skip vs. Fail for Unsupported or Unconfigured Features

Treating a missing prerequisite (kernel config option, hardware feature,
filesystem capability) as a hard failure turns an environment difference
into a false regression signal, and — per documented kselftest policy —
a test that fails outright when unconfigured is also expected to not break
the top-level `make run_tests` run for everyone else.

- If a syscall or ioctl fails with `EOPNOTSUPP`/`ENOSYS`/`ENODEV` because the
  specific feature genuinely isn't present, that's a skip, not a failure:
  call `ksft_test_result_skip()` / the harness `SKIP()` macro, or
  `TEST_REQUIRE()` up front, with a message explaining *what* was missing.
- Distinguish this from the feature being present but broken — that's a
  real failure and must still be reported as one.
- Always include a reason string in the skip message (e.g. "MADV_REMOVE not
  supported by filesystem") — a bare skip with no explanation is nearly as
  unhelpful to a future debugger as a silent pass.

```c
// WRONG: EOPNOTSUPP here means "prerequisite absent", not "test failed"
ret = madvise(addr, len, MADV_REMOVE);
ASSERT_EQ(ret, 0);

// CORRECT: treat the missing capability as a skip, with a reason
ret = madvise(addr, len, MADV_REMOVE);
if (ret == -1 && errno == EOPNOTSUPP)
	SKIP(return, "MADV_REMOVE not supported by filesystem");
ASSERT_EQ(ret, 0);
```

**REPORT as bugs**: a test that asserts/fails on `EOPNOTSUPP`, `ENOSYS`, or
similar "capability absent" errno values instead of skipping, or that skips
silently with no message.

## Reuse Existing Shared Test Libraries Instead of Reimplementing Them

Rewriting namespace setup/teardown, busy-wait polling, sysfs/file I/O, or
device-creation plumbing inside one test file instead of using the
subsystem's existing test-util header produces a second implementation with
different corner-case behavior (e.g. no error check on a short write, or a
namespace leak the shared version already guards against), and a bug fixed
in the shared version won't propagate to the reimplementation. Before
writing a small I/O or setup helper, check whether the subsystem's own test
utility header already has it.

- mm tests: `tools/testing/selftests/mm/vm_util.h` already provides sysfs
  I/O (`read_sysfs`, `write_sysfs`), general file I/O (`read_file`,
  `write_file`, `read_num`, `write_num`), and result reporting
  (`log_test_start`, `log_test_result`). A new mm test that hand-opens a
  sysfs path with `open()`/`write()`/`close()` instead of calling
  `write_sysfs()` is reimplementing something that already exists, usually
  without the existing error handling.
- Networking tests: `tools/testing/selftests/net/lib.sh` already provides
  namespace management (`setup_ns`, `cleanup_ns`, `cleanup_all_ns`),
  busy-wait polling (`busywait`, `busywait_for_counter`, `loopy_wait`),
  structured result reporting (`log_test`, `log_test_result`,
  `log_test_skip`, `handle_test_result_*`, `ksft_status_merge`), and
  netdevsim helpers (`create_netdevsim`, `cleanup_netdevsim`). A new net/
  shell test that hand-rolls any of these is very likely duplicating
  something already hardened against the common failure modes.
- BPF tests: `tools/testing/selftests/bpf/README.rst` documents the
  `DENYLIST` mechanism for excluding tests on architectures that lack a
  feature, and `vmtest.sh` for running under a matched kernel — check there
  before adding an ad hoc per-architecture skip.
- More generally: any subsystem's `tools/testing/selftests/<subsys>/` tree
  tends to have its own `*_util.h`/`lib.sh`/`lib.mk`-style header collecting
  helpers new tests are expected to use — don't assume none exists just
  because a given helper isn't in `net/lib.sh` or `mm/vm_util.h`.

**REPORT as bugs**: a test that hand-rolls sysfs/file I/O, namespace
setup/teardown, or a polling loop instead of using the subsystem's existing
test-util header (`vm_util.h` for mm, `lib.sh` for net, etc.) for something
that header already provides.

## KVM Selftests: IRQ Chip Setup and `vm_create` vs `vm_create_with_one_vcpu`

Tests that use `KVM_IRQFD`, `KVM_IRQ_LINE`, or IRQ routing APIs after
`vm_create()` fail because `vm_create()` does not create vCPUs, and on arm64
VGIC finalization (`KVM_DEV_ARM_VGIC_CTRL_INIT`) requires all vCPUs to be
created first. On architectures without any in-kernel IRQ chip support (riscv,
loongarch), these ioctls fail with `-ENODEV`.

`vm_create(nr_runnable_vcpus)` allocates a VM and sizes memory for the given
number of vCPUs, but does **not** create any vCPUs. IRQ chip setup is
initiated during `vm_create()` via `kvm_arch_vm_post_create()`, but
finalization (via `kvm_arch_vm_finalize_vcpus()`) only happens in functions
that also create vCPUs, such as `vm_create_with_one_vcpu()` and
`__vm_create_with_vcpus()`.

`kvm_arch_has_default_irqchip()` returns whether the architecture sets up an
in-kernel IRQ chip by default:

| Architecture | Return value |
|--------------|-------------|
| x86 | `true` (creates IOAPIC/PIC/LAPIC via `vm_create_irqchip()`) |
| s390 | `true` |
| arm64 | `true` when GICv3 is supported and not disabled via `test_disable_default_vgic()` |
| riscv, loongarch | `false` (weak default in `lib/kvm_util.c`) |

Tests that need an in-kernel IRQ chip must:

1. Call `TEST_REQUIRE(kvm_arch_has_default_irqchip())` to skip on architectures
   that lack IRQ chip support.
2. Use `vm_create_with_one_vcpu()` (or `__vm_create_with_vcpus()`) rather than
   bare `vm_create()`, so that vCPUs are created and IRQ chip finalization
   completes before issuing IRQ-related ioctls.

```c
// WRONG: vm_create() does not create vCPUs or finalize the IRQ chip
vm = vm_create(1);
kvm_irqfd(vm, gsi, eventfd, 0);

// CORRECT: Skip unsupported architectures, then create VM with vCPU
TEST_REQUIRE(kvm_arch_has_default_irqchip());
vm = vm_create_with_one_vcpu(&vcpu, NULL);
kvm_irqfd(vm, gsi, eventfd, 0);
```

## Quick Checks

- **New shared files**: When a commit creates a file that is sourced or
  imported by test scripts, verify it is added to `TEST_FILES` in the Makefile
- **`TEST_PROGS` vs `TEST_FILES`**: Executable tests go in `TEST_PROGS`;
  supporting files go in `TEST_FILES`. Mixing these up causes either execution
  failures or missing installations
- **KVM IRQ chip tests**: When tests use `KVM_IRQFD`, `KVM_IRQ_LINE`, or IRQ
  routing, verify `vm_create_with_one_vcpu()` is used and
  `TEST_REQUIRE(kvm_arch_has_default_irqchip())` is present
- **Hardcoded constants**: flag hardcoded filenames, IPs, or magic numbers
  where a loop/glob/table would cover future additions for free (e.g. a
  hardcoded `*.BTF` filename instead of a `*.BTF` glob in an install rule) —
  ask whether the specific value has a documented rationale or is just the
  first one the author tried
- **`Fixes:` tag accuracy on bugfix patches**: for a patch fixing a bug in
  existing test code, verify the `Fixes:` tag points at the commit that
  actually introduced the bug, not just the commit that added the general
  area of code — this is checked mechanically by CI bots on some lists
  (e.g. bpf) and gets flagged when wrong
