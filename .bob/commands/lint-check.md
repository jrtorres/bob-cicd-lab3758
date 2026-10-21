---
description: Run Checkstyle on order-service and summarize findings by severity with fixes
---
Run a Checkstyle lint check on the order-service module and report the results.

1. Run this command from the workspace root:

   ```
   mvn -f order-service/pom.xml checkstyle:check
   ```

   A non-zero exit code is expected when violations exist. Do not stop on it, and do not try to fix the build. If Maven fails for another reason (for example, the file below was not generated), report that error and stop.

2. Read only `order-service/target/checkstyle-result.xml`. Do not read the Maven console output beyond confirming the command ran, and do not read other report files. Open source files only when needed to write a concrete fix for a specific finding.

3. Group the findings by severity (`error`, then `warning`, then `info`). For each severity, show:
   - The total count.
   - Findings grouped by rule (the `source` attribute, shortened to the check name, such as `UnusedImports`), with the count per rule and the affected `file:line` locations.

4. Name the fix-first items. Rank them by severity first, then by how many occurrences a rule has, then by how much risk it carries (for example, correctness or maintainability issues before whitespace). Give a short list of at most 5 items with a one-line reason for each.

5. For each fix-first item, suggest a concrete remediation. Show the specific change for the offending code (a before/after snippet or the exact edit), not generic advice. Note when a whole rule can be fixed in bulk, such as with an IDE "optimize imports" or formatter run.

Do not modify any files unless the user asks. If the report has no findings, say so in one line.
