# QEMU Maintainer Workflow & Pull Request Guide

This guide describes standard operational workflows for QEMU subsystem maintainers, from reviewing and collecting patches to running tests and submitting signed pull requests.

---

## 1. Patch Collection & Queue Management

Subsystem maintainers are listed in the `MAINTAINERS` file. Patches are received via the `qemu-devel@nongnu.org` mailing list.

### 1.1 Applying Patches with `b4`

The `b4` tool simplifies fetching and applying patch series from public inboxes (e.g. `lore.kernel.org/qemu-devel/`):

```bash
# Fetch and apply a patch series by Message-Id:
b4 shazam <Message-Id>

# Or fetch mbox:
b4 am <Message-Id>
git am <mbox-file>
```

`b4` automatically gathers `Reviewed-by:`, `Acked-by:`, and `Tested-by:` follow-ups from the thread and records the original `Message-Id:`.

### 1.2 Inspecting Maintainership

Verify which files fall under your subsystem or who needs to be CC'd:

```bash
./scripts/get_maintainer.pl <patch-file>
# or for a commit:
./scripts/get_maintainer.pl -f <path/to/file.c>
```

---

## 2. Testing & Verification

Before merging patches into a maintainer branch, run comprehensive test suites:

### 2.1 Code Style Checks
```bash
./scripts/checkpatch.pl <patch-file>
# Or check the local branch commits:
./scripts/checkpatch.pl --branch master..HEAD
```

### 2.2 Unit & QTest Suites
```bash
# Fast unit tests:
make check-unit

# Subsystem QTest integration tests:
make check-qtest

# Specific architecture or device test:
make check-qtest-x86_64
make check-qtest-aarch64 SPEED=slow
```

### 2.3 Functional Tests
```bash
# Run Python-based functional tests:
make check-functional
```

### 2.4 Sanitizer Builds (ASan / UBSan)
Test device models for memory safety regressions:
```bash
../configure --enable-sanitizers --enable-debug
make -j$(nproc)
make check
```

---

## 3. Preparing and Submitting Pull Requests

QEMU subsystem maintainers submit pull requests to the project coordinator (Peter Maydell) and `qemu-devel@nongnu.org`.

### 3.1 Creating a Signed Git Tag

All QEMU pull requests require a cryptographically signed tag:

```bash
# Format: <subsystem>-pull-<YYYYMMDD> or similar
git tag -s -m "Subsystem queue for YYYY-MM-DD" my-subsystem-20260903 HEAD
git push origin my-subsystem-20260903
```

### 3.2 Generating the Pull Request

Use `git request-pull` to format the request:

```bash
git request-pull master https://gitlab.com/<user>/qemu.git my-subsystem-20260903 > /tmp/pull.txt
```

### 3.3 Pull Request Email Structure

Send the pull request as `[PULL 00/NN]` using `git format-patch --cover-letter --subject-prefix=PULL`, pasting the `git request-pull` output into the cover letter. Retransmit the individual patches as threaded follow-ups (`git send-email --thread`) to the cover letter:

```text
Subject: [PULL 00/12] <Subsystem> queue YYYY-MM-DD

The following changes since commit <base-commit-hash>:

  <base commit subject> (YYYY-MM-DD)

are available in the Git repository at:

  https://gitlab.com/<username>/qemu.git tags/my-subsystem-YYYYMMDD

for you to fetch changes up to <top-commit-hash>:

  <top commit subject> (YYYY-MM-DD)

----------------------------------------------------------------
<Subsystem name> queue:
- Brief bulleted summary of features and fixes included in this pull
- Major bug fixes (citing CVEs or GitLab issue numbers)
----------------------------------------------------------------

<Diffstat output from git request-pull>

<List of commits with authors and subjects>
```

---

## 4. GitLab CI Integration

- Push the maintainer branch to your GitLab fork to trigger the full CI pipeline (`.gitlab-ci.d/`).
- Ensure cross-architecture builds (s390x, ppc64le, Windows/MSYS2, macOS) pass before sending the pull request.
