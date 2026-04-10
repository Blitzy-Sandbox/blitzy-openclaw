# Blitzy Project Guide

---

## 1. Executive Summary

### 1.1 Project Overview

This project addresses a critical `spawn sh ENOENT` bug in the OpenClaw shell execution subsystem. The `getShellConfig()` function in `src/agents/shell-utils.ts` returned a relative shell binary name `"sh"` when `process.env.SHELL` was undefined, causing `child_process.spawn()` to fail with ENOENT whenever a custom `env` object overrode the child process's PATH. The fix resolves the shell binary to an absolute filesystem path using the parent process's PATH, with a POSIX-standard `/bin/sh` fallback. This is a targeted, single-file bug fix affecting agent shell execution on Linux, macOS, CI environments, and Docker containers.

### 1.2 Completion Status

```mermaid
pie title Completion Status
    "Completed (6h)" : 6
    "Remaining (2h)" : 2
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 8 |
| **Completed Hours (AI)** | 6 |
| **Remaining Hours** | 2 |
| **Completion Percentage** | 75.0% |

**Calculation:** 6 completed hours / (6 completed + 2 remaining) = 6 / 8 = **75.0%**

### 1.3 Key Accomplishments

- [x] Root cause identified: relative `"sh"` fallback in `getShellConfig()` at `shell-utils.ts:48` fails when child process receives custom `env.PATH`
- [x] Fix implemented: shell binary resolved to absolute path via `resolveShellFromPath("sh") ?? "/bin/sh"`
- [x] Additional safety guard added: `envShell.startsWith("/")` rejects bogus `process.env.SHELL` values
- [x] Targeted test verified: `bash-tools.exec.path.test.ts` — 2/2 tests passed (previously 1 failing)
- [x] Full regression suite passed: 906 test files, 5,974 tests, zero failures
- [x] TypeScript build verified: `pnpm build` completed with zero errors
- [x] Lint verification passed: `oxlint --type-aware` — 0 warnings, 0 errors

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| No critical unresolved issues | N/A | N/A | N/A |

All AAP-specified code changes and verifications are complete. No blocking issues remain.

### 1.5 Access Issues

No access issues identified. All repository operations, test suites, and build commands executed successfully during autonomous validation.

### 1.6 Recommended Next Steps

1. **[High]** Human code review of the 2-file diff (12 lines added, 2 removed) — verify the absolute path resolution logic and `startsWith("/")` guard
2. **[High]** Merge to main branch and trigger CI pipeline to confirm green build in production CI environment
3. **[Medium]** Verify fix behavior in Docker/CI environments where `process.env.SHELL` is unset
4. **[Low]** Monitor production logs for any residual `spawn ENOENT` errors post-deployment

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Root Cause Diagnosis & Analysis | 1.5 | Traced ENOENT to relative `"sh"` in `getShellConfig()` line 48; analyzed spawn behavior with custom `env.PATH`; cross-referenced with `resolveShell()` in `shell-env.ts` |
| Fix Implementation (shell-utils.ts) | 1.0 | Replaced `"sh"` with `resolveShellFromPath("sh") ?? "/bin/sh"`; added `envShell.startsWith("/")` guard; added 7-line rationale comment |
| Test File Adjustment (shell-utils.test.ts) | 0.5 | Updated test expectation from `"sh"` to `"/bin/sh"` to match new absolute path fallback behavior |
| Targeted Test Verification | 0.5 | Ran `bash-tools.exec.path.test.ts` and `shell-utils.test.ts` — confirmed 6/6 tests pass |
| Full Regression Testing | 1.5 | Executed full suite: 906 test files, 5,974 tests across unit, extensions, and gateway configs — all passed |
| Build & Lint Verification | 1.0 | Ran `pnpm build` (TypeScript compilation clean) and `oxlint --type-aware` (0 warnings, 0 errors) |
| **Total Completed** | **6.0** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human Code Review & Approval | 1.0 | High |
| CI Pipeline Validation on Merge | 0.5 | High |
| Production Deployment Verification | 0.5 | Medium |
| **Total Remaining** | **2.0** | |

---

## 3. Test Results

All test data below originates from Blitzy's autonomous validation logs for this project.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Unit Tests | Vitest (vitest.unit.config.ts) | 4,906 | 4,906 | 0 | N/A | 800 test files; includes shell-utils.test.ts (4 tests) |
| Extensions Tests | Vitest (vitest.extensions.config.ts) | 876 | 876 | 0 | N/A | 73 test files |
| Gateway Tests | Vitest (vitest.gateway.config.ts) | 192 | 192 | 0 | N/A | 33 test files; includes bash-tools.exec.path.test.ts (2 tests) |
| **Total** | **Vitest v4.0.18** | **5,974** | **5,974** | **0** | **N/A** | **906 test files, 100% pass rate** |

**Key Test Results:**
- `bash-tools.exec.path.test.ts` > `merges login-shell PATH for host=gateway` — ✅ PASSED
- `bash-tools.exec.path.test.ts` > `skips login-shell PATH when env.PATH is provided` — ✅ PASSED (previously FAILING)
- `shell-utils.test.ts` > `getShellConfig` — ✅ 4/4 PASSED (including updated expectation for `/bin/sh`)

---

## 4. Runtime Validation & UI Verification

### Runtime Health

- ✅ `getShellConfig()` returns absolute shell path (`/usr/bin/sh`) when `process.env.SHELL` is unset
- ✅ Shell binary spawns successfully with custom `env.PATH` overrides
- ✅ `pnpm build` compiles without errors — TypeScript strict mode satisfied
- ✅ `oxlint --type-aware src/agents/shell-utils.ts` — 0 warnings, 0 errors
- ✅ Git working tree clean — no uncommitted changes

### UI Verification

- N/A — This is a backend shell utility fix with no UI components

### API Integration

- ✅ `createExecTool({ host: "gateway" })` functions correctly with explicit `env.PATH` parameter
- ✅ `createExecTool({ host: "gateway" })` functions correctly with default PATH merging (no regression)

---

## 5. Compliance & Quality Review

| AAP Requirement | Status | Evidence |
|----------------|--------|----------|
| Modify line 48 of `shell-utils.ts` — resolve to absolute path | ✅ Pass | Diff confirms `resolveShellFromPath("sh") ?? "/bin/sh"` replacement |
| Add rationale comment above the change | ✅ Pass | 7-line comment added at lines 48–54 |
| No new files created | ✅ Pass | `git diff --stat` shows 0 new files |
| No new dependencies added | ✅ Pass | `pnpm-lock.yaml` unchanged |
| No new tests added | ✅ Pass | Existing test expectation updated (justified scope expansion) |
| Targeted test passes (`bash-tools.exec.path.test.ts`) | ✅ Pass | 2/2 tests passed |
| Full regression suite passes | ✅ Pass | 5,974/5,974 tests passed |
| Build integrity verified | ✅ Pass | `pnpm build` SUCCESS |
| No modifications outside bug fix scope | ✅ Pass | Only `shell-utils.ts` and `shell-utils.test.ts` modified |
| Follows repository coding conventions | ✅ Pass | TypeScript, ESM, ternary with nullish coalescing |

### Fixes Applied During Validation

| Fix | File | Description |
|-----|------|-------------|
| Absolute shell path resolution | `shell-utils.ts:55-58` | Replaced `"sh"` with `resolveShellFromPath("sh") ?? "/bin/sh"` |
| Bogus SHELL value guard | `shell-utils.ts:56` | Added `envShell.startsWith("/")` check to reject non-absolute SHELL values |
| Test expectation update | `shell-utils.test.ts:80` | Changed `expect(shell).toBe("sh")` to `expect(shell).toBe("/bin/sh")` |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Absolute shell path not found on exotic system | Technical | Low | Low | POSIX `/bin/sh` fallback ensures availability on all standard systems | Mitigated |
| `process.env.SHELL` set to non-absolute path | Technical | Low | Low | `startsWith("/")` guard added; falls through to `resolveShellFromPath()` | Mitigated |
| Regression in fish shell handling | Technical | Medium | Very Low | Fish branch (lines 38–46) untouched; uses same `resolveShellFromPath()` function | Mitigated |
| Windows platform affected | Integration | Low | None | Windows branch returns early at line 23 — modified code unreachable on Windows | Not Applicable |
| CI environment has different PATH | Operational | Low | Low | Fix resolves using parent `process.env.PATH` at config time, not child PATH | Mitigated |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 6
    "Remaining Work" : 2
```

**Hours Distribution:**
- **Completed Work:** 6 hours (75.0%) — Root cause analysis, fix implementation, test adjustment, full regression testing, build and lint verification
- **Remaining Work:** 2 hours (25.0%) — Human code review, CI validation, production deployment verification

---

## 8. Summary & Recommendations

### Achievement Summary

The project has achieved **75.0% completion** (6 of 8 total hours). All autonomous coding, testing, and validation work specified in the Agent Action Plan has been delivered successfully. The root cause — a relative `"sh"` fallback in `getShellConfig()` — was identified, fixed, and verified across the full test suite of 5,974 tests with zero failures.

### Key Metrics

| Metric | Value |
|--------|-------|
| Files Modified | 2 |
| Lines Added | 12 |
| Lines Removed | 2 |
| Net Change | +10 lines |
| Tests Passing | 5,974 / 5,974 (100%) |
| Build Status | Clean |
| Lint Status | Clean |

### Remaining Gaps

The remaining 2 hours (25.0%) consist entirely of human review and deployment activities:
1. **Code review** (1h) — A senior developer should review the 2-file diff to validate the absolute path resolution approach and the `startsWith("/")` guard
2. **CI pipeline** (0.5h) — Merge to main triggers CI; verify all tests pass in the production CI environment
3. **Deployment verification** (0.5h) — Confirm fix resolves the issue in Docker/CI environments where `SHELL` is unset

### Production Readiness Assessment

The fix is **production-ready** pending human code review. The change is minimal (1 expression modified, 1 guard added, 1 test expectation updated), uses existing internal functions (`resolveShellFromPath`), follows established patterns (`resolveShell()` in `shell-env.ts`), and has been validated against the full test suite with 100% pass rate.

### Recommendations

1. **Merge promptly** — This is a blocker-severity bug affecting any agent shell execution with custom `env.PATH` on systems where `SHELL` is unset
2. **No additional testing required** — The existing test suite comprehensively covers the fix scenario
3. **Consider backporting** — If older release branches are maintained, this single-line fix should be backported

---

## 9. Development Guide

### System Prerequisites

| Component | Required Version | Verification Command |
|-----------|-----------------|---------------------|
| Node.js | >= 22.12.0 | `node -v` |
| pnpm | 10.23.0 | `pnpm -v` |
| Git | Any recent version | `git --version` |
| OS | Linux or macOS (POSIX) | `uname -a` |

### Environment Setup

```bash
# Clone the repository
git clone <repository-url>
cd openclaw

# Checkout the fix branch
git checkout blitzy-722b1f93-d29e-4575-8da4-00c49c251fe9

# Install dependencies
pnpm install
```

### Verify the Fix

```bash
# Run the targeted test (the previously failing test)
npx vitest run src/agents/bash-tools.exec.path.test.ts --reporter=verbose

# Expected output:
# ✓ merges login-shell PATH for host=gateway
# ✓ skips login-shell PATH when env.PATH is provided
# Tests  2 passed (2)
```

### Run Shell Utils Tests

```bash
# Run shell-utils unit tests
npx vitest run src/agents/shell-utils.test.ts --reporter=verbose

# Expected output:
# ✓ prefers bash when fish is default and bash is on PATH
# ✓ falls back to sh when fish is default and bash is missing
# ✓ falls back to env shell when fish is default and no sh is available
# ✓ uses sh when SHELL is unset
# Tests  4 passed (4)
```

### Build the Project

```bash
# TypeScript compilation
pnpm build

# Expected: Clean build with no errors
```

### Run Full Test Suite

```bash
# Full regression test
pnpm test

# Expected: 906 test files, 5,974 tests, 0 failures
```

### Lint Verification

```bash
# Run linter on the modified file
npx oxlint --type-aware src/agents/shell-utils.ts

# Expected: 0 warnings, 0 errors
```

### Verify Absolute Shell Resolution (Manual)

```bash
# Confirm getShellConfig() returns absolute path
node --import tsx -e "
  import { getShellConfig } from './src/agents/shell-utils.js';
  const config = getShellConfig();
  console.log('Shell:', config.shell);
  console.log('Is absolute:', config.shell.startsWith('/'));
"

# Expected output:
# Shell: /usr/bin/sh (or similar absolute path)
# Is absolute: true
```

### Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| `pnpm: command not found` | pnpm not installed globally | Run `npm install -g pnpm@10.23.0` |
| Node.js version mismatch | Requires >= 22.12.0 | Use `nvm install 22` and `nvm use 22` |
| Tests hang in watch mode | vitest defaults to watch | Always use `npx vitest run` (not `npx vitest`) |
| `ERR_MODULE_NOT_FOUND` on import | Dependencies not installed | Run `pnpm install` first |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `pnpm install` | Install all dependencies |
| `pnpm build` | TypeScript compilation |
| `pnpm test` | Run full test suite (parallel) |
| `npx vitest run <file> --reporter=verbose` | Run specific test file |
| `npx oxlint --type-aware <file>` | Lint specific file |
| `git diff origin/blitzy-49b968aa-00d6-4679-a553-ad9bd7d4960c...blitzy-722b1f93-d29e-4575-8da4-00c49c251fe9` | View full diff |

### B. Port Reference

No ports are used by this fix. The change is to an internal shell utility function with no network components.

### C. Key File Locations

| File | Purpose | Status |
|------|---------|--------|
| `src/agents/shell-utils.ts` | Shell config utility — contains the fix | MODIFIED |
| `src/agents/shell-utils.test.ts` | Unit tests for shell config | MODIFIED |
| `src/agents/bash-tools.exec.ts` | Exec tool implementation (calls `getShellConfig()`) | UNCHANGED |
| `src/agents/bash-tools.exec.path.test.ts` | PATH handling integration tests | UNCHANGED |
| `src/infra/shell-env.ts` | Shell environment resolution (reference pattern) | UNCHANGED |

### D. Technology Versions

| Technology | Version |
|------------|---------|
| Node.js | >= 22.12.0 (runtime: v22.22.2) |
| pnpm | 10.23.0 |
| TypeScript | Strict ESM (target: es2023) |
| Vitest | 4.0.18 |
| oxlint | Latest (via npx) |
| OpenClaw | 2026.1.30 |

### E. Environment Variable Reference

| Variable | Relevance | Notes |
|----------|-----------|-------|
| `SHELL` | Primary shell path source | When set and absolute, used directly by `getShellConfig()` |
| `PATH` | Shell binary resolution fallback | Used by `resolveShellFromPath()` to locate `sh` |

### G. Glossary

| Term | Definition |
|------|------------|
| ENOENT | "Error No Entity" — OS error when a file or executable cannot be found |
| `getShellConfig()` | Function in `shell-utils.ts` that returns the shell binary path and arguments for spawning |
| `resolveShellFromPath()` | Internal function that searches `process.env.PATH` for an executable by name and returns its absolute path |
| POSIX | Portable Operating System Interface — standard defining `/bin/sh` as the default shell location |
| `child_process.spawn()` | Node.js API for spawning child processes; uses `options.env.PATH` for command lookup when custom env is provided |