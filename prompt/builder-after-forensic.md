SCOPE: fix only what the Forensic report identifies. Do not touch anything else.

--- FORENSIC REPORT ---
[PASTE FORENSIC REPORT HERE]
--- END FORENSIC REPORT ---

Work through the failures in the order Forensic listed them.
For each failure:
  1. Read the file Forensic identified before touching it
  2. Apply exactly the fix Forensic specified — no broader refactoring
  3. Run the verification Forensic listed under "Risk" if one is implied
  4. Move to the next failure

Rules:
- Cascading failures (where Forensic says "resolve Failure N, no independent fix needed") — skip them, they resolve automatically
- Do not change any file not mentioned in the Forensic report
- Do not modify spec files, feature files, or AGENTS.md
- Do not run the full test suite — that is Tester's job
- After all fixes are applied, run: go build ./... from each affected service directory and confirm it exits 0

When all fixes are applied and builds pass, output exactly:
"Builder done. Ready for Tester."
