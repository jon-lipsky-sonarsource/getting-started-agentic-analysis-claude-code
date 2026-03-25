# /sonar-scan

Fetch SonarQube coding guidelines for the files I'm currently working on,
then run advanced code analysis on all recently modified files, and report
a summary of findings.

## Steps

1. Identify the files modified in this session (or ask the user if unclear).

2. Call `get_guidelines` with:
   - `mode: "combined"` to get both catalog guidelines and project-specific issues
   - `file_paths`: the list of modified files
   - `languages`: inferred from file extensions (e.g. `["java"]` for `.java` files)

3. For each modified file, call `run_advanced_code_analysis` with:
   - `filePath`: the project-relative path to the file
   - `branchName`: the current git branch (run `git branch --show-current` if needed)
   - `fileScope`: `["MAIN"]` for production code, `["TEST"]` for test files

4. Aggregate all issues across all analysed files.

5. Report a summary:
   - Total issue count by severity (BLOCKER, CRITICAL, MAJOR, MINOR, INFO)
   - For each issue: file, line, rule key, message, and a brief explanation
   - Overall pass/fail status (pass = zero BLOCKER/CRITICAL/MAJOR issues)

6. If any BLOCKER or CRITICAL issues are found, offer to fix them immediately.

## Notes

- Run this skill after completing a logical unit of work (e.g., a new method,
  a new class, or a significant refactoring).
- Use `/sonar-verify` for a quicker check that skips the guidelines fetch.
- Use `/sonar-review` for a full pre-PR review with fix suggestions.
