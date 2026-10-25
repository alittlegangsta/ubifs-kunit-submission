# UBIFS node scanning KUnit tests

## Assignment Objective

Provide small, repeatable unit tests for the real Linux UBIFS node scanner, without a development board or flash device. The suite exercises `ubifs_scan_a_node()` using in-memory inputs and the kernel's own KUnit framework.

**Delivery status:** the local interview package is complete. Patch application, source-tree equivalence, a fresh build and two complete test runs passed. A format-compatible official checkpatch revision reports 0 errors and one explained MAINTAINERS warning. The pinned Base checker still reports its different message-format diagnostics; both complete results are included. The signed commit and patch remain unchanged. This interview package has been published to this GitHub
repository but has not been submitted upstream.

## Linux Base Commit

- Official source: https://github.com/torvalds/linux.git
- Base: `734bef548a96e8774d6eef5dfa2026c6edc4e80b`
- Human-reviewed development commit: `f39a9d9ad451a583f6d1fbedd3b669cd82c57324`
- Source tree: `35767453ea08c237d07d3563beb6c510f699a03e`

The development commit has exactly one parent: the Base above. Do not replace the Base with the development commit or the Base of a reference patch.

## Patch Overview

[The patch](patches/0001-ubifs-add-KUnit-coverage-for-node-scanning.patch) contains exactly three files, 210 insertions, and no deletions:

| File | Change |
|---|---|
| `fs/ubifs/Kconfig` | Add `UBIFS_KUNIT_TEST`, requiring `KUNIT=y` and `UBIFS_FS=y` (+13 lines). |
| `fs/ubifs/Makefile` | Link `scan_test.o` into the existing UBIFS composite object (+1). |
| `fs/ubifs/scan_test.c` | Add one suite, eight cases, managed allocations and minimal context/header helpers (+196). |

No production function, symbol export, or filesystem I/O implementation is changed. Project documents and logs are outside the kernel patch. The new test source uses `SPDX-License-Identifier: GPL-2.0`.

## Test Cases

Suite: `ubifs-scan`.

| Case | Expected behavior |
|---|---|
| `ubifs_scan_empty_space_test` | Erased prefix → `SCANNED_EMPTY_SPACE`. |
| `ubifs_scan_bad_magic_test` | Invalid magic/non-padding → `SCANNED_GARBAGE`. |
| `ubifs_scan_padding_bytes_test` | Eight padding bytes followed by non-padding → 8. |
| `ubifs_scan_bad_padding_bytes_test` | Seven padding bytes → `SCANNED_GARBAGE`. |
| `ubifs_scan_short_header_test` | Valid magic with insufficient header length → `SCANNED_GARBAGE`. |
| `ubifs_scan_valid_node_test` | Valid commit-start node → `SCANNED_A_NODE`. |
| `ubifs_scan_bad_crc_test` | Valid control, then changed CRC → `SCANNED_A_CORRUPT_NODE`; CRC error counter becomes 1. |
| `ubifs_scan_bad_padding_node_test` | Valid PAD control, then invalid alignment with a recomputed CRC → `SCANNED_A_BAD_PAD_NODE`. |

## Test Environment

Validated on Ubuntu 24.04.4 LTS / WSL2 x86_64, GCC 13.3.0, GNU Make 4.3, Python 3.12.3, Git 2.43.0 and QEMU 8.2.2. The tested kernel is built from the Base plus this patch; it does not replace the host kernel. Standard kernel build dependencies include flex, bison, OpenSSL and ELF development headers, and bc.

QEMU used TCG after KVM/HVF were unavailable. `acpi=off` is a guest-only setting retained from the validated baseline to avoid that environment's ACPI initialization errors. Neither real MTD/UBI devices nor a mounted UBIFS filesystem are required.

## How to Apply the Patch

Start in this package directory. Use an existing official Linux Git clone containing the exact Base. Set `UBIFS_LINUX` to its absolute path. A shallow clone of today's HEAD might not contain this older commit; obtain the exact commit before proceeding. The commands below use the same worktree layout and arguments validated for this delivery, with the package and repository locations supplied by the recipient.

```bash
UBIFS_PACKAGE="$PWD"
UBIFS_LINUX=/absolute/path/to/linux
UBIFS_BASE=$(cat "$UBIFS_PACKAGE/BASE_COMMIT.txt")
UBIFS_WORKTREE="$UBIFS_PACKAGE/../worktrees/stage-06-patch-apply"
UBIFS_BUILD="$UBIFS_PACKAGE/../builds/stage-06-patch-apply"

git -C "$UBIFS_LINUX" cat-file -e "$UBIFS_BASE^{commit}"
# Both destinations must be unused. If either check fails, choose new paths.
test ! -e "$UBIFS_WORKTREE" && test ! -e "$UBIFS_BUILD"
# Continue only if the checks above succeeded.
git -C "$UBIFS_LINUX" worktree add --quiet --detach "$UBIFS_WORKTREE" "$UBIFS_BASE"
git -C "$UBIFS_WORKTREE" rev-parse HEAD
git -C "$UBIFS_WORKTREE" status --short
git -C "$UBIFS_WORKTREE" am -- "$UBIFS_PACKAGE/patches/0001-ubifs-add-KUnit-coverage-for-node-scanning.patch"
git -C "$UBIFS_WORKTREE" rev-parse HEAD^ 'HEAD^{tree}'
```

Before `git am`, HEAD must equal the Base and status must be empty. Afterwards the parent must equal the Base and the tree must equal the tree listed above. Git needs your own committer identity for `git am`; no identity is supplied or changed by this package. `git am` preserves the existing author/message/sign-off from the patch. Its new commit SHA may differ because committer metadata differs. If application fails, preserve the worktree and inspect the error instead of forcing or cleaning it.

## How to Run KUnit

Use the same shell variables. This is the validated invocation, with absolute path values adapted to the recipient's workspace. `results.json` is a chosen output name, not an input dependency. For the first run the build directory must be new.

```bash
cd "$UBIFS_WORKTREE"
python3 tools/testing/kunit/kunit.py run \
  --arch=x86_64 --jobs=8 --timeout=120 --kernel_args=acpi=off \
  --build_dir="$UBIFS_BUILD" \
  --kunitconfig="$UBIFS_PACKAGE/config/ubifs.kunitconfig" \
  --json="$UBIFS_BUILD/results.json" \
  ubifs-scan
```

Check the command's exit status, all eight names, and `Ran 8 tests: passed: 8`. A successful compilation or zero discovered cases is insufficient. Preserve the JSON and `$UBIFS_BUILD/test.log` before a repeat; change the JSON filename for the repeat. The six-option [configuration](config/ubifs.kunitconfig) is included; no developer-local config is needed. Eight jobs were suitable on the tested host with about 14 GiB available memory; reduce concurrency if needed.

## Validation Results

Actual validation on 2026-10-24:

| Check | Result |
|---|---|
| `git format-patch --base=<Base> -1 <commit>` | Exit 0; exactly the three expected files and correct Base metadata. |
| `git am` in a clean detached Base worktree | Exit 0; applied commit `5ffe8d4a0c228fcf81df2038c8e75b6d0d1402fa`. |
| Applied tree vs. development tree | Identical; original development HEAD and working tree unchanged. |
| Fresh build and all target tests | 8 PASS, 0 failures/skips/crashes/errors; exit 0; 50.860 s total. |
| Repeat with the same source/config | 8 PASS, 0 failures/skips/crashes/errors; exit 0; 3.534 s total. |
| Committed diff whitespace check | Exit 0. |
| Full patch, pinned Base checker | **1 error, 2 warnings, exit 1; retained, not a clean Base-checker result.** |
| Full patch, official `2a8d68338ee77e200895f2697c56fd00094767f7` checker | **0 errors, 1 MAINTAINERS warning, exit 1.** |
| Source-only diff checkpatch | 0 errors, 1 warning, exit 1. |

The version difference is now traced to official source history. [Commit `2a8d68338ee77e200895f2697c56fd00094767f7`](https://github.com/torvalds/linux/commit/2a8d68338ee77e200895f2697c56fd00094767f7), committed on August 5, accepts a nonempty free-form Assisted-by value and skips email validation for that tag. Its unmodified checker was downloaded into a separate tools directory and run against this exact patch.

A later [commit `1e035297ffdadf2184c3c773b8c098e372324aaf`](https://github.com/torvalds/linux/commit/1e035297ffdadf2184c3c773b8c098e372324aaf), committed on October 14, removes the tag support. That script is byte-for-byte identical to the project's official Base script and local script. Consequently, the Base is **later**, not an old version missing the August fix. It treats Assisted-by as non-standard and parses LLM as an email address. The accessed online AI guide still shows the August-style tag, whereas the pinned checkout differs. The August checker is a historical format-compatible checker, not a newer replacement for Base. The comparison does not erase the Base diagnostics or establish current upstream acceptance.

Both runs used default checks, an explicitly empty `.checkpatch.conf`, the unchanged kernel tree and the same full mail-format patch; no `--ignore`, `--types`, `--no-signoff`, filtering or patch edits were used. Tool revisions, hashes and results are recorded in [tool-compatibility.json](logs/tool-compatibility.json). The author's instruction is to preserve the signed commit and disclose this version difference; no source or runtime defect was found that prevents interview delivery.

The remaining `FILE_PATH_CHANGES` warning is triggered by adding `fs/ubifs/scan_test.c`. In the Base, the UBI FILE SYSTEM (UBIFS) section already contains `F: fs/ubifs/` at line 28137, covering the test. No extra MAINTAINERS entry is necessary. The warning is retained and makes the compatible checker's exit status 1, despite zero errors.

[Validation data](logs/validation.json), [first KTAP excerpt](logs/kunit-first.ktap), [repeat excerpt](logs/kunit-repeat.ktap), [Base checker output](logs/checkpatch-base.txt), and [compatible checker output](logs/checkpatch-compatible.txt) are included. The earlier [checkpatch output](logs/checkpatch.txt) is preserved too. KTAP files are exact test-block excerpts, not entire boot logs; their corresponding full-console hashes are recorded. Full commands, console logs and resolved configs remain outside this small package. Tool compatibility review re-audited the existing application and test evidence against identical source/config; it did not rerun KUnit.

Earlier stage-5 results on identical source included two detected mutations with restored PASS, Generic KASAN, and a KUnit-disabled production link. These are historical checks, not rerun or represented as fresh sanitizer results here.

## Known Limitations

Coverage is deliberately limited: not all node types, variable-length ranges, padding boundaries, nonzero offsets, data-node CRC policies, recovery, mounts, flash I/O, modules, or other architectures are tested. UBSan was not run. The Base checker retains the documented message-format diagnostics. Do not describe this package as passing every checker version or as accepted upstream. If a recipient mandates zero errors specifically from the pinned Base checker, discuss that requirement explicitly; the historical checker does not satisfy such a mandate.

## AI Assistance Disclosure

Codex (LLM) assisted with implementation, source analysis, review, test execution, documentation and commit-message drafting. The human author reviewed the contribution and personally created the original commit and its Signed-off-by. AI did not add a sign-off, amend that commit, or publish this package. Human certification and AI disclosure serve different purposes.

After the eight tests and quality checks were complete, the project compared the [September 11 cover letter](https://lkml.iu.edu/hypermail/linux/kernel/2609.1/09437.html) and [scan-test patch](https://lkml.iu.edu/hypermail/linux/kernel/2609.1/09439.html). Available records did not show prior explicit use; that does not prove absolute independent originality. This retrospective review did not apply or copy reference code. The reference Base `df2908090cda368b01ff43709f51890076c56157` is not this package's Base. Reference-author test claims are not our validation results.

## Package Integrity

[FILELIST.txt](FILELIST.txt) lists every delivery file, including itself and SHA256SUMS. [SHA256SUMS](SHA256SUMS) covers every file except itself. From this directory, run:

```bash
sha256sum --check SHA256SUMS
```

Only this package's contents are intended for an interview repository. Do not upload the full Linux checkout, build outputs, private project records or the temporary checker download directory. This package is publicly available in this GitHub repository.
It has not been submitted to the Linux upstream project.
