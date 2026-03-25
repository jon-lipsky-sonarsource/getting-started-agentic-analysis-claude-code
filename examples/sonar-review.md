# /sonar-review

Full pre-PR SonarQube review. Fetches coding guidelines, runs analysis on all
changed files, summarises all issues by severity, suggests specific fixes
using Sonar rule documentation, and asks whether to auto-fix.

Use this skill immediately before opening a pull request.

## Steps

1. Determine the base branch (default: `main`). Ask the user if unclear.

2. Identify all files changed relative to the base branch:
   ```
   git diff --name-only <base-branch>...HEAD
   ```

3. Infer the languages from the file extensions of changed files.

4. Call `get_guidelines` with:
   - `mode: "combined"`
   - `file_paths`: all changed source files
   - `languages`: inferred language list

5. For each changed source file, call `run_advanced_code_analysis`.

6. Aggregate all issues. Group by severity: BLOCKER → CRITICAL → MAJOR → MINOR → INFO.

7. Print a structured report:
   ```
   ## Pre-PR SonarQube Review

   Files analysed: N
   Total issues: N (N blocker, N critical, N major, N minor, N info)

   ### BLOCKER (must fix before merge)
   - src/.../File.java:45  java:SXXXX  Description of issue
     Fix: [specific fix suggestion based on rule documentation]

   ### CRITICAL
   ...

   ### MAJOR
   ...

   ### MINOR / INFO (review at your discretion)
   ...

   ## Summary
   [READY TO MERGE / NOT READY — reason]
   ```

8. If there are any BLOCKER or CRITICAL issues, ask:
   "Would you like me to fix all BLOCKER and CRITICAL issues now?"
   - If yes: fix each issue, then re-run analysis to confirm the fixes are clean.
   - If no: provide the list for manual remediation.

9. If MAJOR issues exist, ask: "Would you like me to fix MAJOR issues as well?"

10. After all fixes are applied (or declined), print the final pass/fail status.

## Notes

- This skill is intentionally thorough and may take a few minutes on large
  changesets. Use `/sonar-verify` for a quicker mid-development check.
- The auto-fix step re-runs analysis after fixes to confirm correctness.
  It does not blindly apply fixes — it verifies each one.
- MINOR and INFO issues are reported but not offered for auto-fix by default,
  as they represent style preferences rather than correctness concerns.
