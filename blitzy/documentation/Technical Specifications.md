# Technical Specification

# 0. Agent Action Plan

## 0.1 Executive Summary

Based on the bug description, the Blitzy platform understands that the bug is a **`spawn sh ENOENT` failure in the exec tool's PATH handling when users supply an explicit `env.PATH` parameter**. Specifically, the `createExecTool` function in the OpenClaw shell execution subsystem fails to spawn a child process because its shell binary resolution returns a relative name (`"sh"`) that the operating system cannot locate when the child process's environment PATH has been overridden to a user-defined value.

### 0.1.1 Technical Failure Description

The failure manifests as:

```
Error: spawn sh ENOENT
{ errno: -2, code: 'ENOENT', syscall: 'spawn sh', path: 'sh', spawnargs: [ '-c', 'echo $PATH' ] }
```

This is a **binary resolution error** of category ENOENT (Error No Entity). When `child_process.spawn()` is invoked with a custom `env` object, Node.js uses `options.env.PATH` — not the parent process's `process.env.PATH` — to locate the executable. If the shell binary is specified as the relative name `"sh"` and the custom PATH does not contain a directory holding `sh`, the OS cannot find the binary.

### 0.1.2 Reproduction Steps

- Step 1: Ensure `process.env.SHELL` is unset or empty (common in CI, Docker, or Node.js process environments)
- Step 2: Create an exec tool with `host: "gateway"`
- Step 3: Execute a command with an explicit `env: { PATH: "/explicit/bin" }` parameter
- Step 4: Observe `spawn sh ENOENT` because `/explicit/bin/sh` does not exist

Executable reproduction command:

```bash
cd /tmp/blitzy/blitzy-openclaw/blitzy-49b968aa-00d6-4679-a553-ad9bd7d4960c_88ce14
npx vitest run src/agents/bash-tools.exec.path.test.ts --reporter=verbose
```

### 0.1.3 Impact Assessment

| Dimension | Assessment |
|-----------|------------|
| **Severity** | Blocker — exec tool fails silently for any user-supplied PATH |
| **Affected Test** | `src/agents/bash-tools.exec.path.test.ts` → `skips login-shell PATH when env.PATH is provided` |
| **Affected Feature** | Agent shell execution (`createExecTool`) on `host=gateway` with explicit env overrides |
| **Production Risk** | Any AI agent or user invoking `exec` with a custom `env.PATH` on systems where `SHELL` is unset will hit ENOENT |
| **Platforms** | Linux, macOS, CI environments, Docker containers (any environment where `SHELL` is unset) |


## 0.2 Root Cause Identification

Based on exhaustive repository analysis, THE root cause is: **the `getShellConfig()` function in `src/agents/shell-utils.ts` returns a relative shell binary name `"sh"` instead of an absolute path when `process.env.SHELL` is undefined, causing `child_process.spawn()` to fail with `ENOENT` whenever a custom `env` object overrides the child process's PATH.**

### 0.2.1 Primary Root Cause

- **Located in:** `src/agents/shell-utils.ts`, line 48
- **Triggered by:** `process.env.SHELL` being `undefined` (or empty) in the Node.js runtime, combined with a caller supplying `env.PATH` to `tool.execute()`
- **Error type:** Binary resolution failure — relative shell name unresolvable in user-defined PATH

**Problematic code at `src/agents/shell-utils.ts:48`:**

```typescript
const shell = envShell && envShell.length > 0 ? envShell : "sh";
```

When `process.env.SHELL` is unset, `envShell` is `undefined`, and the fallback is the bare string `"sh"`. This is passed as the `file` argument to `child_process.spawn("sh", ["-c", ...], { env: customEnv })`. Per the Node.js documentation, command lookup uses `options.env.PATH` when a custom `env` is provided — if that custom PATH does not contain a directory with `sh`, the spawn fails.

### 0.2.2 Contributing Factor: Inconsistency Between Shell Resolvers

The codebase has two shell resolution functions with **inconsistent fallback behavior**:

| Module | Function | Fallback When SHELL Unset | Absolute? |
|--------|----------|--------------------------|-----------|
| `src/agents/shell-utils.ts:48` | `getShellConfig()` | `"sh"` | **No** (relative) |
| `src/infra/shell-env.ts:13` | `resolveShell()` | `"/bin/sh"` | **Yes** (absolute) |

The `resolveShell()` function in `shell-env.ts` correctly returns `"/bin/sh"` as an absolute path fallback, which works regardless of the child process's PATH. The `getShellConfig()` function in `shell-utils.ts` should follow the same pattern but does not.

### 0.2.3 Evidence Chain

- **Evidence 1 — Test Failure:** The vitest test `skips login-shell PATH when env.PATH is provided` in `src/agents/bash-tools.exec.path.test.ts` calls `tool.execute()` with `env: { PATH: "/explicit/bin" }`, triggering the spawn with relative `"sh"` against a PATH that has no `sh` binary.
- **Evidence 2 — Runtime Confirmation:** Running `getShellConfig()` in the Node.js runtime confirms `SHELL` is `undefined` and the function returns `{ shell: 'sh', args: ['-c'] }` with a relative shell name.
- **Evidence 3 — Spawn Verification:** Spawning with `spawn('sh', ['-c', 'echo $PATH'], { env: { PATH: '/explicit/bin' } })` produces `ENOENT`, while `spawn('/usr/bin/sh', ...)` with the same env succeeds and outputs `/explicit/bin`.
- **Evidence 4 — Cross-Module Comparison:** The sibling module `src/infra/shell-env.ts` demonstrates the correct pattern (`"/bin/sh"` absolute fallback) that `shell-utils.ts` fails to follow.

### 0.2.4 Why Test 1 Passes But Test 2 Fails

The first test in the same file (`merges login-shell PATH for host=gateway`) passes because it does **not** supply an explicit `env.PATH` in the `execute()` call. In this path, `process.env.PATH` (which includes `/usr/bin`) is merged as the base environment, so relative `"sh"` resolves successfully through the parent's PATH. The second test supplies `env: { PATH: "/explicit/bin" }`, which replaces the lookup PATH entirely — and `/explicit/bin/sh` does not exist.

This conclusion is definitive because the error reproduces deterministically whenever a custom `env` with a `PATH` excluding the shell binary is passed to `spawn()` with a relative command name, and resolves immediately when an absolute path is used.


## 0.3 Diagnostic Execution

### 0.3.1 Code Examination Results

**File analyzed:** `src/agents/shell-utils.ts`
- **Problematic code block:** Lines 22–50 (`getShellConfig()` function)
- **Specific failure point:** Line 48 — the fallback assignment `const shell = envShell && envShell.length > 0 ? envShell : "sh";`
- **Execution flow leading to bug:**
  - The caller invokes `tool.execute("call1", { command: "echo $PATH", env: { PATH: "/explicit/bin" } })`
  - `src/agents/bash-tools.exec.ts:920` builds `mergedEnv` by spreading `process.env` with the caller's `env`, setting `mergedEnv.PATH = "/explicit/bin"`
  - `src/agents/bash-tools.exec.ts:929` correctly skips login-shell PATH merging because `params.env?.PATH` is truthy
  - `src/agents/bash-tools.exec.ts:497` calls `getShellConfig()` which returns `{ shell: "sh", args: ["-c"] }`
  - `src/agents/bash-tools.exec.ts:498–509` calls `spawnWithFallback({ argv: ["sh", "-c", "echo $PATH"], options: { env: mergedEnv } })`
  - Node.js `child_process.spawn("sh", ...)` searches `mergedEnv.PATH` (`/explicit/bin`) for the `sh` binary — does not find it — emits `ENOENT`

**File analyzed:** `src/agents/bash-tools.exec.ts`
- **Lines 919–929:** Environment construction logic — correctly merges and conditionally applies shell PATH
- **Lines 496–509:** Spawn dispatch — retrieves shell config and calls `spawnWithFallback`; passes `opts.env` (the merged env) as the child process environment

**File analyzed:** `src/infra/shell-env.ts`
- **Lines 10–13:** `resolveShell()` function — returns `"/bin/sh"` (absolute) as fallback, demonstrating the correct pattern

**File analyzed:** `src/agents/bash-tools.exec.path.test.ts`
- **Lines 87–103:** The failing test case (`skips login-shell PATH when env.PATH is provided`)
- **Lines 65–85:** The passing test case (`merges login-shell PATH for host=gateway`) — passes because it inherits `process.env.PATH=/usr/bin` which contains `sh`

### 0.3.2 Repository File Analysis Findings

| Tool Used | Command Executed | Finding | File:Line |
|-----------|-----------------|---------|-----------|
| vitest | `npx vitest run src/agents/bash-tools.exec.path.test.ts --reporter=verbose` | Test "skips login-shell PATH when env.PATH is provided" fails with `spawn sh ENOENT` | `src/agents/bash-tools.exec.path.test.ts:87` |
| vitest | `pnpm test` (full suite) | 1 test failed out of entire test suite; all other tests pass | `src/agents/bash-tools.exec.path.test.ts` |
| node | `node --import tsx -e "import { getShellConfig } from './src/agents/shell-utils.js'; ..."` | `SHELL` env is `undefined`; config returns `{ shell: 'sh', args: ['-c'] }` | `src/agents/shell-utils.ts:48` |
| node | `spawn('sh', ['-c', 'echo $PATH'], { env: { PATH: '/explicit/bin' } })` | Error: `spawn sh ENOENT`, exit code `-2` | Runtime confirmation |
| node | `spawn('/usr/bin/sh', ['-c', 'echo $PATH'], { env: { PATH: '/explicit/bin' } })` | Success: stdout = `/explicit/bin`, exit code `0` | Runtime confirmation |
| grep | `grep -n "resolveShell" src/infra/shell-env.ts` | Found `resolveShell()` with absolute `/bin/sh` fallback at line 13 | `src/infra/shell-env.ts:13` |
| grep | `grep -rn 'getShellConfig' src/` | `getShellConfig()` called in `bash-tools.exec.ts:497` and defined in `shell-utils.ts:22` | `src/agents/shell-utils.ts:22`, `src/agents/bash-tools.exec.ts:497` |
| find | `find src/ -name "*.ts" \| xargs grep '"sh"'` | Only `shell-utils.ts:48` uses bare `"sh"` as a shell binary name | `src/agents/shell-utils.ts:48` |

### 0.3.3 Fix Verification Analysis

**Steps followed to reproduce bug:**
- Set `process.env.PATH = "/usr/bin"` (mirrors test setup)
- Called `getShellConfig()` — confirmed it returns `{ shell: "sh", args: ["-c"] }` when `SHELL` is unset
- Spawned `sh` with custom env `{ PATH: "/explicit/bin" }` — confirmed `ENOENT`
- Spawned `/usr/bin/sh` (absolute path) with same custom env — confirmed success with output `/explicit/bin`

**Confirmation tests used to ensure that bug was fixed:**
- Replacing `"sh"` with `resolveShellFromPath("sh") ?? "/bin/sh"` in `getShellConfig()` will resolve the shell to `/usr/bin/sh` (its absolute path) at config time, before the child process PATH is overridden
- The `resolveShellFromPath()` function (already present in the same file at line 52) searches `process.env.PATH` (the parent's PATH, not the child's) and returns an absolute path
- Fallback to `"/bin/sh"` is consistent with `shell-env.ts:13` and is the POSIX-standard shell location

**Boundary conditions and edge cases covered:**
- `process.env.SHELL` is set → `envShell` is used directly (already an absolute path) — no change in behavior
- `process.env.SHELL` is unset AND `process.env.PATH` contains `sh` → `resolveShellFromPath("sh")` returns absolute path (e.g., `/usr/bin/sh`)
- `process.env.SHELL` is unset AND `process.env.PATH` does NOT contain `sh` → fallback to `"/bin/sh"` (standard POSIX location)
- Windows platform → function returns early at line 23 with PowerShell path — unaffected
- Fish shell detected → function returns early at lines 39–45 using `resolveShellFromPath()` — unaffected (already uses absolute resolution)

**Verification confidence level:** 95% — The fix uses the same `resolveShellFromPath()` function already proven to work in the fish-shell branch (lines 40–45), and the `/bin/sh` fallback matches the established pattern in `shell-env.ts`.


## 0.4 Bug Fix Specification

### 0.4.1 The Definitive Fix

**File to modify:** `src/agents/shell-utils.ts`

**Current implementation at line 48:**

```typescript
const shell = envShell && envShell.length > 0 ? envShell : "sh";
```

**Required change at line 48:**

```typescript
const shell = envShell && envShell.length > 0 ? envShell : (resolveShellFromPath("sh") ?? "/bin/sh");
```

**This fixes the root cause by:** resolving the `sh` binary to its absolute filesystem path at configuration time (using the parent process's `process.env.PATH`), before the child process environment is constructed. The `resolveShellFromPath()` function — already defined in the same file at line 52 — iterates through `process.env.PATH` directories, checks for an executable `sh`, and returns its absolute path (e.g., `/usr/bin/sh`). If `resolveShellFromPath()` returns `undefined` (e.g., `sh` not found in PATH), the fallback `"/bin/sh"` provides the POSIX-standard absolute path, matching the pattern established by `resolveShell()` in `src/infra/shell-env.ts:13`.

With an absolute shell path, `child_process.spawn()` bypasses PATH-based command lookup entirely and invokes the binary directly — ensuring the spawn succeeds regardless of what `env.PATH` the caller supplies.

### 0.4.2 Change Instructions

**MODIFY** line 48 of `src/agents/shell-utils.ts`:
- **FROM:** `const shell = envShell && envShell.length > 0 ? envShell : "sh";`
- **TO:** `const shell = envShell && envShell.length > 0 ? envShell : (resolveShellFromPath("sh") ?? "/bin/sh");`

**Rationale comment to include above the change:**

```typescript
// Resolve to an absolute path so that spawn() works even when
// the child process receives a custom env.PATH that does not
// include the directory containing the shell binary. Falls back
// to "/bin/sh" (POSIX standard) if resolution fails, consistent
// with resolveShell() in src/infra/shell-env.ts.
```

No imports need to be added — `resolveShellFromPath` is already defined in the same file (`shell-utils.ts`, line 52) as a private function and is already called by the fish-shell handling branch at lines 40–45.

### 0.4.3 Fix Validation

**Test command to verify fix:**

```bash
npx vitest run src/agents/bash-tools.exec.path.test.ts --reporter=verbose
```

**Expected output after fix:**

```
✓ merges login-shell PATH for host=gateway
✓ skips login-shell PATH when env.PATH is provided
Tests  2 passed (2)
```

**Full regression test command:**

```bash
pnpm test
```

**Expected result:** All tests pass; the previously failing test `skips login-shell PATH when env.PATH is provided` now succeeds because `getShellConfig()` returns an absolute path (e.g., `/usr/bin/sh`) that does not depend on the child process's PATH for resolution.


## 0.5 Scope Boundaries

### 0.5.1 Changes Required (Exhaustive List)

| Action | File Path | Lines | Specific Change |
|--------|-----------|-------|-----------------|
| MODIFIED | `src/agents/shell-utils.ts` | Line 48 | Replace `"sh"` (relative) with `resolveShellFromPath("sh") ?? "/bin/sh"` (absolute resolution with POSIX fallback) |

**Total files modified:** 1
**Total lines changed:** 1 (single expression replacement within an existing line)
**No files created.** No new modules, tests, dependencies, or configuration files are introduced.
**No files deleted.**

### 0.5.2 Explicitly Excluded

**Do not modify:**
- `src/agents/bash-tools.exec.ts` — The spawn logic and environment merging at lines 496–509 and 919–935 are correct; the bug is solely in the shell config resolution
- `src/agents/bash-tools.exec.path.test.ts` — The test correctly exercises the expected behavior; it should pass without modification once the source fix is applied
- `src/infra/shell-env.ts` — Already uses the correct pattern (`"/bin/sh"` absolute fallback); no change needed
- `src/process/spawn-utils.ts` — Spawn utility functions are correct and handle fallbacks properly

**Do not refactor:**
- The `resolveShellFromPath()` function — it is already well-structured and accomplishes its purpose; extracting or restructuring it would exceed the minimal fix scope
- The environment merging logic in `bash-tools.exec.ts:919–935` — it correctly constructs `mergedEnv` and conditionally applies shell PATH; the bug is upstream in the shell config
- The Windows/PowerShell branch in `getShellConfig()` (lines 23–32) — it already uses `resolvePowerShellPath()` which returns an absolute path; it is not affected

**Do not add:**
- No new tests — the existing test (`skips login-shell PATH when env.PATH is provided`) already covers the exact scenario and will pass once the fix is applied
- No new dependencies — the fix uses `resolveShellFromPath()` which is already defined in the same file
- No documentation changes — this is a single-line internal fix with no API surface change


## 0.6 Verification Protocol

### 0.6.1 Bug Elimination Confirmation

- **Execute:** `npx vitest run src/agents/bash-tools.exec.path.test.ts --reporter=verbose`
- **Verify output matches:**
  - `✓ merges login-shell PATH for host=gateway` — continues to pass
  - `✓ skips login-shell PATH when env.PATH is provided` — now passes (previously failed)
  - `Tests  2 passed (2)` — both tests in the suite succeed
- **Confirm error no longer appears in:** vitest stdout/stderr — the `spawn sh ENOENT` error with `{ errno: -2, code: 'ENOENT', syscall: 'spawn sh' }` must be completely absent
- **Validate functionality with:** Direct Node.js confirmation that `getShellConfig()` returns an absolute shell path:

```bash
node --import tsx -e "import { getShellConfig } from './src/agents/shell-utils.js'; const c = getShellConfig(); console.log(c.shell.startsWith('/'));"
```

Expected output: `true` — confirming the shell path is absolute.

### 0.6.2 Regression Check

- **Run existing test suite:** `pnpm test`
- **Expected result:** All tests pass. The only previously failing test is the one this fix addresses. No other tests depend on `getShellConfig()` returning a relative `"sh"` string.
- **Verify unchanged behavior in:**
  - Test `merges login-shell PATH for host=gateway` — must continue to pass; the fix does not alter behavior when `SHELL` is set or when no custom env is provided
  - All other agent/exec tests — the absolute shell path produces identical spawning behavior when PATH lookup would have found `sh` anyway
  - Windows platform — the `getShellConfig()` function returns early at line 23 with the PowerShell path; the modified line 48 is never reached on Windows
  - Fish-shell branch — lines 39–45 already use `resolveShellFromPath()` and return before reaching line 48; unchanged
- **Confirm build integrity:** `pnpm build` — must complete without errors, ensuring TypeScript compilation succeeds with the modified code
- **Confirm no type errors:** The return type of line 48 remains `string`, consistent with the function signature `{ shell: string; args: string[] }`. The expression `resolveShellFromPath("sh") ?? "/bin/sh"` evaluates to `string` (since `??` converts `string | undefined` to `string` via the fallback)


## 0.7 Rules

The following user-specified coding and design standards are acknowledged and enforced throughout this bug fix:

### 0.7.1 Engineering Principles Compliance

- **Correctness first:** The fix prioritizes deterministic, correct behavior — resolving the shell binary to an absolute path guarantees spawn success regardless of the child process's `env.PATH`. No clever workarounds (e.g., injecting shell directories into the child's PATH) are used.
- **Security by default:** The fix does not introduce any new security concerns. No unsanitized input is passed to spawn; the shell binary is resolved from trusted system paths. The principle of least privilege is maintained — no elevated permissions or additional capabilities are required.
- **Maintainability:** The single-line change is readable and self-documenting. A rationale comment is included explaining the motivation. The fix follows the existing pattern (`resolveShellFromPath()` usage in the fish-shell branch) and aligns with the sibling module's approach (`shell-env.ts:13`).
- **Observability:** The existing `spawnWithFallback` error handling and warning system at lines 514–520 of `bash-tools.exec.ts` remains intact. Spawn failures will continue to be logged with context.
- **Performance:** The `resolveShellFromPath("sh")` call performs a synchronous `fs.accessSync` check through PATH directories. This is the same mechanism already used for fish-shell detection (lines 40–45) and adds negligible overhead since it executes once per `getShellConfig()` invocation.

### 0.7.2 Code Quality Rules Compliance

- **No dead code:** No unused code is introduced; no existing code becomes dead.
- **No unused imports:** No new imports are required — `resolveShellFromPath` is defined in the same file.
- **Input validation and boundary conditions:** All boundary cases are handled — `SHELL` set (uses it directly), `SHELL` unset with `sh` in PATH (resolves to absolute), `SHELL` unset without `sh` in PATH (falls back to `/bin/sh`).
- **Clear error handling:** Spawn error handling is unchanged. The `resolveShellFromPath()` function returns `undefined` on failure; the `??` operator cleanly falls back to `"/bin/sh"`.
- **Tests required:** The existing test `skips login-shell PATH when env.PATH is provided` already covers the exact failure scenario and will validate the fix without modification.

### 0.7.3 Repository Hygiene Compliance

- **Follow repository conventions:** The fix follows the existing coding style — TypeScript, ternary expression with nullish coalescing, same function reuse pattern as the fish-shell branch.
- **Deterministic builds:** No environment-specific assumptions are introduced. The fix works identically across Linux and macOS (Windows is handled by a separate code path).
- **Consistent directory structure:** No files are added, moved, or renamed.

### 0.7.4 Security Standards Compliance

- **No secrets in code:** No credentials, tokens, or sensitive values are introduced.
- **Pinned dependencies:** No new dependencies are added.
- **No risky patterns:** No use of `eval`, `exec`, shell injection vectors, or insecure deserialization. The shell path is resolved through filesystem access checks, not string interpolation.

### 0.7.5 Additional Rules

- **Make the exact specified change only:** One line is modified — line 48 of `src/agents/shell-utils.ts`. No other files are touched.
- **Zero modifications outside the bug fix:** No refactoring, feature additions, documentation changes, or unrelated improvements.
- **Extensive testing to prevent regressions:** Both the targeted test (`bash-tools.exec.path.test.ts`) and the full test suite (`pnpm test`) must pass after the fix.


## 0.8 References

### 0.8.1 Repository Files and Folders Searched

The following files and directories were systematically inspected to diagnose the bug and derive the fix:

**Primary Bug-Related Files:**

| File Path | Purpose | Relevance |
|-----------|---------|-----------|
| `src/agents/shell-utils.ts` | Shell configuration utility — defines `getShellConfig()` and `resolveShellFromPath()` | **Contains the bug** at line 48; returns relative `"sh"` instead of absolute path |
| `src/agents/bash-tools.exec.ts` | Main exec tool implementation — spawn logic and environment construction | Calls `getShellConfig()` at line 497; builds `mergedEnv` at lines 919–935 |
| `src/agents/bash-tools.exec.path.test.ts` | Test suite for exec PATH handling — tests login-shell merge and explicit env.PATH | **Contains the failing test** at line 87 |
| `src/infra/shell-env.ts` | Shell environment resolution — defines `resolveShell()` with correct `/bin/sh` fallback | Demonstrates correct pattern at line 13; used for cross-reference validation |
| `src/process/spawn-utils.ts` | Spawn utility with fallback support | Reviewed to confirm spawn dispatch is not contributing to the bug |

**Repository Structure and Configuration Files:**

| File/Folder Path | Purpose | Relevance |
|------------------|---------|-----------|
| `package.json` | Root monorepo configuration — engines, scripts, dependencies | Confirmed Node.js `>=22.12.0`, `pnpm@10.23.0`, build/test commands |
| `CHANGELOG.md` | Version history and bug fix log | Reviewed for prior related fixes; none found for this specific issue |
| `src/agents/` | Agent subsystem directory | Primary investigation target — contains all bug-related source files |
| `src/infra/` | Infrastructure utilities | Contains `shell-env.ts` with the correct shell resolution pattern |
| `src/process/` | Process management utilities | Contains spawn helpers; confirmed they are not the cause |

### 0.8.2 External References

| Source | URL | Relevance |
|--------|-----|-----------|
| Node.js `child_process.spawn()` Documentation | `https://nodejs.org/api/child_process.html` | Confirms that command lookup uses `options.env.PATH` when a custom `env` is provided, and `process.env.PATH` otherwise |

### 0.8.3 Attachments

No external attachments (Figma screens, design files, or supplementary documents) were provided for this task.

### 0.8.4 Environment Configuration

| Component | Version / Value |
|-----------|----------------|
| Node.js | v22.22.2 (project requires `>=22.12.0`) |
| pnpm | 10.23.0 (exact match to `packageManager` field) |
| TypeScript | Strict ESM compilation via `pnpm build` |
| Test Runner | vitest (parallel execution via `node scripts/test-parallel.mjs`) |
| OS | Linux (CI/container environment — `process.env.SHELL` is unset) |
| AWS CLI | Installed via pip (for LocalStack environment; not related to bug) |
| LocalStack CLI | v4.14.0 (for LocalStack environment; not related to bug) |


