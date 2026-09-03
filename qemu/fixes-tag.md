# QEMU Commit Tags & Fixes Verification

Commit messages in QEMU convey vital provenance, issue tracking, and backport metadata. Maintainers and reviewers must verify that all tags follow standard formats.

---

## 1. Commit Tags Reference

### 1.1 `Fixes:` Tag

Identifies the commit that introduced a bug or regression.

- **Format**: `Fixes: <12+ character SHA-1> ("Original Commit Subject")`
- **Rules**:
  1. Minimum 12 hex characters for the commit hash.
  2. The subject line must match the original commit's first line exactly and be enclosed in double quotes.
  3. Must remain on a **single line** (never wrapped, even if exceeding 80 columns).
  4. Must appear in the trailer block (above the `---` separator).
- **Example**:
  ```text
  Fixes: 9a78bf4e21d3 ("hw/arm/virt: Support up to 512 vCPUs")
  ```
- **Verification Checklist**:
  - Run `git cat-file -t <hash>` to ensure the commit exists.
  - Run `git log -1 --format=%s <hash>` to verify the subject matches.
  - Verify that the referenced commit is genuinely the root cause of the bug.

---

### 1.2 `Resolves:` Tag

QEMU's official issue tracker is hosted on GitLab. When a patch closes an issue:

- **Format**: `Resolves: https://gitlab.com/qemu-project/qemu/-/issues/<number>`
- **Example**:
  ```text
  Resolves: https://gitlab.com/qemu-project/qemu/-/issues/1842
  ```
- **External Bug Trackers**: Use `Buglink: https://...` for Launchpad, Red Hat Bugzilla, or Debian bug trackers.

---

### 1.3 `Cc: qemu-stable@nongnu.org`

Indicates that the patch should be backported to the current active QEMU stable maintenance releases.

- **Criteria**:
  - The patch fixes a crash, host hang, security vulnerability, or serious regression.
  - The patch is small, low-risk, and self-contained.
  - It does NOT introduce new features or change user-visible command-line / QMP syntax.
- **Placement**: Placed in the trailer block alongside `Signed-off-by:`.
- **Dependency Notes**: If the fix depends on a prerequisite commit:
  ```text
  Cc: qemu-stable@nongnu.org # v8.2+: commit abc1234 ("prerequisite fix")
  ```

---

### 1.4 Provenance & Attribution Tags

- **`Signed-off-by:`**: Mandatory Developer Certificate of Origin (DCO). Every author, co-developer, and maintainer handling the patch must add their S-o-b.
- **`Reviewed-by:`**: Added by reviewers who audited the patch.
- **`Acked-by:`**: Subsystem maintainer approval without necessarily doing a line-by-line review.
- **`Tested-by:`**: Confirms the patch was tested on specific hardware or workloads.
- **`Reported-by:`**: Credits the person who found and reported the issue.
- **`Suggested-by:`**: Credits the person who suggested the solution.
- **`Co-developed-by:`**: Credits joint authors (must be immediately followed by that author's `Signed-off-by:`).
- **`Message-Id:`**: Added by maintainers when applying patches from `qemu-devel` using `b4` or `git am`.

---

## 2. Commit Subject Line Format

- **Subsystem Prefix**: Every commit subject must indicate the subsystem or path touched:
  - `hw/arm/virt: Fix interrupt routing for SPIs`
  - `block/qcow2: Handle truncated images gracefully`
  - `target/i386: Fix CPUID leaf 0x80000008`
  - `migration: Prevent memory leak on cancel`
  - `qapi: Deprecate obsolete parameter 'foo'`
- **Imperative Mood**: Use imperative sentences ("Fix ...", "Add ...", "Refactor ...", not "Fixed" or "Adds").
- **Length**: Keep under 72 characters if possible, never exceeding 78 characters.
