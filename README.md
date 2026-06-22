# Contribution [1]: `containsAtMain` does not handle `@main` on the same line as a multi-line end comment

**Contribution Number:** [1]  
**Student:** [Ramya Lakshmi Kuppa Sundararajan]  
**Issue:** [GitHub issue link](https://github.com/swiftlang/swift-package-manager/issues/9685)  
**Status:** [Phase II] [Complete]

---

## Why I Chose This Issue

The `containsAtMain` function in `SPMBuildCore` fails to detect `@main` when it appears on the same line as the end of a block comment (`*/`). This impacts Swift package configuration detection. I chose this issue because it is a specific, well-contained bug that will allow me to practice Swift string parsing and testing within the SPM codebase. Also, it has a clear success criteria (the unit test passing).

---

## Understanding the Issue

### Problem Description

The `containsAtMain` function in `MainAttrDetection.swift` parses Swift source files line-by-line to detect the `@main` attribute. It tracks whether the current line is inside a multi-line block comment using a `multilineComment` boolean. The bug is in how it detects the end of a comment: it checks `line.hasSuffix("*/")`, which only matches when `*/` is at the very end of the line. When a line reads `*/ @main` — closing a comment and immediately declaring `@main` — the suffix check fails, `multilineComment` stays `true`, and the entire line is skipped, so `@main` is never detected.

### Expected Behavior

When `@main` appears after the closing `*/` of a block comment on the same line (e.g., `*/ @main`), `containsAtMain` should return `true`, recognising `@main` as a valid entry point declaration outside the comment.

### Current Behavior

`containsAtMain` returns `false` for files where `@main` follows `*/` on the same line. The function incorrectly treats the remainder of the line as still being inside the comment block.

### Affected Components

- `Sources/SPMBuildCore/MainAttrDetection.swift` — contains the buggy implementation of `containsAtMain`
- `Tests/SPMBuildCoreTests/MainAttrDetectionTests.swift` — contains the failing test case (marked `withKnownIssue`) that demonstrates the bug, and also had a typo (`/* @main` instead of `*/ @main`) in the test file content

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment — challenges you faced, how you solved them]

### Steps to Reproduce

1. Run `swift test --filter SPMBuildCoreTests.MainAttrDetectionTests/containsAtMainReturnsExpectedValueCurrentlyFails`
2. Observe the output: `recorded a known issue … Expectation failed: (actual → false) == (expected → true)`
3. The function returns `false` for a file containing `*/ @main`, when it should return `true`

### Reproduction Evidence

- **Commit showing reproduction:** [fix-issue-containsAtMain branch](https://github.com/RamyaLakshmiKS/swift-package-manager/tree/fix-issue-containsAtMain-does-not-handle-%40main-on-the-same-line-as-a-multi-line-end-comment)
- **My findings:** The existing test suite already had a `withKnownIssue`-wrapped test case for this bug at `MainAttrDetectionTests.swift:245`. Running it confirms that `containsAtMain` returns `false` instead of `true`. I also identified a typo in the test: the file content used `/* @main` (an opening delimiter) where `*/ @main` (a closing delimiter) was intended, consistent with the issue title "end comment".

---

## Solution Approach

### Analysis

The root cause is in `MainAttrDetection.swift` at the multiline comment tracking logic. The original code used two separate checks:

```swift
if line.hasSuffix("*/") {
    multilineComment = false
}
if multilineComment {
    continue
}
```

`hasSuffix("*/")` only returns `true` when `*/` is the last thing on the line. A line like `*/ @main` ends with `@main`, so the suffix check fails, `multilineComment` remains `true`, and the `continue` skips the line entirely — including the `@main` that follows the comment close.

### Proposed Solution

Replace `hasSuffix("*/")` with `range(of: "*/")`, which finds `*/` anywhere on the line. When found, close the comment and inspect the text that follows `*/` on the same line for `@main`.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `containsAtMain` must correctly detect `@main` that appears after `*/` on the same line, because `*/` closes the comment and everything after it is live code.

**Match:** The function already processes lines one at a time using `hasPrefix` and `hasSuffix`. The fix extends this same pattern by using `range(of:)` instead of `hasSuffix`, which is a standard Swift `String` API consistent with the rest of the function.

**Plan:**
1. In `MainAttrDetection.swift`, replace the `hasSuffix("*/")` block with a `range(of: "*/")` search inside the `if multilineComment` guard, extracting and trimming the substring after `*/` and checking it for `@main`.
2. In `MainAttrDetectionTests.swift`, fix the typo in the known-issue test case (`/* @main` → `*/ @main`).
3. Move the corrected test case from `containsAtMainReturnsExpectedValueCurrentlyFails` (the `withKnownIssue` block) into the main parameterised test `containsAtMainReturnsExpectedValue`, and delete the now-redundant `withKnownIssue` test.

**Implement:** [fix-issue-containsAtMain branch](https://github.com/RamyaLakshmiKS/swift-package-manager/tree/fix-issue-containsAtMain-does-not-handle-%40main-on-the-same-line-as-a-multi-line-end-comment)

**Review:**
- All 16 existing test cases continue to pass
- No new compiler warnings introduced
- Code style matches the surrounding function (no added comments, consistent indentation)
- Fix is minimal — only the broken logic is changed

**Evaluate:** Run `swift test --filter SPMBuildCoreTests.MainAttrDetectionTests` and confirm all test cases pass with no known issues reported.

---

## Testing Strategy

### Unit Tests

- [x] `*/ @main` on the same line as the closing block comment delimiter → `containsAtMain` returns `true`
- [x] `/* @main */` (block comment opening and closing on one line with `@main` inside) → returns `false`
- [x] All 14 pre-existing test cases continue to pass unchanged

### Integration Tests

- [ ] Build a Swift package that uses `*/ @main` and verify SPM correctly identifies the entry point target

### Manual Testing

Ran `swift test --filter SPMBuildCoreTests.MainAttrDetectionTests` after applying the fix. All 16 test cases passed in 0.001 seconds with zero known issues, zero failures.

---

## Implementation Notes

### Code Changes

- **Files modified:**
  - `Sources/SPMBuildCore/MainAttrDetection.swift`
  - `Tests/SPMBuildCoreTests/MainAttrDetectionTests.swift`
- **Key commits:** [fix-issue-containsAtMain branch](https://github.com/RamyaLakshmiKS/swift-package-manager/tree/fix-issue-containsAtMain-does-not-handle-%40main-on-the-same-line-as-a-multi-line-end-comment)
- **Approach decisions:** Used `range(of: "*/")` rather than a full comment-depth counter (which would be needed for nested comments) because the existing function is intentionally a heuristic approximation, not a full parser — the comment at line 16 of the original file acknowledges this. Keeping the fix minimal and consistent with the existing approach is appropriate.

---

## Pull Request

**PR Link:** [swiftlang/swift-package-manager#10211](https://github.com/swiftlang/swift-package-manager/pull/10211)

**PR Description:**

Fix `containsAtMain` to correctly detect `@main` when it appears on the same line as a closing block comment delimiter (`*/`).

**Root cause:** The function used `hasSuffix("*/")` to detect the end of a multi-line comment. This fails for lines like `*/ @main` where `*/` is not the last token. As a result, `multilineComment` remained `true` and the line was skipped entirely.

**Fix:** Replace `hasSuffix("*/")` with `range(of: "*/")` so that `*/` is detected anywhere on the line. The text following `*/` is then checked for `@main`.

**Tests:** Corrected the existing `withKnownIssue` test (which also had a typo — `/*` instead of `*/`) and promoted it to the main parameterised test suite. All 16 test cases pass.

Resolves #9685.

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

- Understanding how `containsAtMain` works as a heuristic line-by-line parser for detecting Swift entry points
- Using Swift's `range(of:)` and substring APIs for mid-string searching
- Working with the Swift Testing framework's `withKnownIssue` and parameterised `arguments:` test patterns
- Navigating a large open source Swift codebase (SPM) to identify the right files and understand existing test conventions

### Challenges Overcome

- The existing failing test had a typo (`/* @main` instead of `*/ @main`) which initially made the bug scenario harder to reason about — tracing through the logic manually clarified that the test file content did not match the issue description.

### What I'd Do Differently Next Time

[Your personal reflection]

---

## Resources Used

- [GitHub Issue #9685](https://github.com/swiftlang/swift-package-manager/issues/9685)
- [`MainAttrDetection.swift` source](Sources/SPMBuildCore/MainAttrDetection.swift)
- [`MainAttrDetectionTests.swift` source](Tests/SPMBuildCoreTests/MainAttrDetectionTests.swift)
- [Swift Testing documentation — `withKnownIssue`](https://developer.apple.com/documentation/testing/withknownissue(_:isintermittent:fileid:filepath:line:column:_:)-30kgk)
