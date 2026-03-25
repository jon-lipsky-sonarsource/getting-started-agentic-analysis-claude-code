# /sonar-verify

Run SonarQube advanced code analysis on the current staged or modified files
and report a pass/fail result with issue details.

This is a lightweight verification check — it skips the guidelines fetch and
goes straight to analysis. Use it frequently during development as a quick
"did I break anything?" gate.

## Steps

1. Identify modified files:
   - Run `git diff --name-only HEAD` to find unstaged changes
   - Run `git diff --name-only --cached` to find staged changes
   - Combine and deduplicate the two lists

2. If no modified files are found, report "No modified files to verify."

3. For each modified file that is a source file (not config, markdown, etc.):
   - Call `run_advanced_code_analysis` with:
     - `filePath`: the project-relative path
     - `branchName`: the current git branch
     - `fileScope`: `["MAIN"]` for production code, `["TEST"]` for test files

4. Report the result for each file:
   ```
   ✓ PASS — src/main/java/example/MyClass.java (0 issues)
   ✗ FAIL — src/main/java/example/OtherClass.java (2 issues)
     [MAJOR] Line 45: java:S1172 — Remove this unused method parameter 'x'.
     [MINOR] Line 12: java:S116  — Rename this field to match the regex '^[a-z][a-zA-Z0-9]*$'.
   ```

5. Final status line:
   - PASS: "All N file(s) passed analysis. No new issues."
   - FAIL: "N file(s) have issues. Fix before committing."

## Notes

- This skill does not fetch guidelines. If you want guidelines context first,
  use `/sonar-scan` instead.
- For a full pre-PR review with fix suggestions, use `/sonar-review`.
- A clean result here matches what CI would report. There is no "it works on
  my machine" exception — the same analysis engine runs in both places.
