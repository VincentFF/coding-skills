---
name: markdown-check
description: Validates Markdown with global Prettier and markdownlint-cli2 immediately before Markdown is committed to Git or written to a remote system such as GitHub, Confluence, or Jira. Do not use for ordinary local drafting or intermediate edits.
compatibility: Requires globally available `prettier` and `markdownlint-cli2` commands.
---

# Markdown Check Gate

## When to Invoke

Invoke this skill **only at a delivery boundary**:

- immediately before committing Markdown files to a Git repository;
- immediately before creating or updating remote Markdown content, including GitHub issues, pull requests, discussions, wikis, or README content; Confluence pages; Jira descriptions or comments; and other non-local systems.

Do **not** invoke this skill while drafting, making intermediate local edits, or merely previewing a local Markdown file. This keeps the authoring loop fast.

For a Git commit, invoke it only if the commit includes `.md` or `.mdx` files. For remote delivery, invoke it for the exact Markdown body or source document that will be sent.

## Preconditions

1. Confirm the required commands are available:

   ```bash
   command -v prettier
   command -v markdownlint-cli2
   ```

2. If either command is unavailable, do not claim validation succeeded. Report the missing command and ask the user to repair their `PATH` or install the tool.
3. Respect repository Prettier and markdownlint configuration when it exists. Do not create, loosen, or replace project lint configuration merely to make a delivery pass.

## Select the Validation Target

### Before a Git Commit

Validate the staged Markdown files, because they are the files that will actually be committed:

```bash
git diff --cached --name-only --diff-filter=ACMR -- '*.md' '*.mdx'
```

- If this returns no files, no Markdown validation is needed.
- If a Markdown file was changed but is not staged, either stage the intended version first or validate that version explicitly and re-run the staged-file check before committing.

### Before a Remote Write

- Prefer the local Markdown file that is the source of the remote body.
- If the body exists only in memory, write the exact proposed body to a temporary `.md` file, validate it, perform the remote write, then remove the temporary file.
- Validate before converting Markdown to a target-specific format. If the final payload is not Markdown (for example, Confluence Storage XHTML), use the target-specific validation rules as well.

## Required Checks

Run both checks against only the selected Markdown files. Quote each path so spaces and shell metacharacters are safe.

```bash
markdownlint-cli2 -- "<file-1>" "<file-2>"
prettier --check "<file-1>" "<file-2>"
```

For a remote body staged in a temporary file, use that temporary file as the target.

- `markdownlint-cli2` enforces the applicable Markdown structural/style rules.
- `prettier --check` parses and verifies formatting consistency.
- A passing result from these tools does **not** prove that links resolve or that every remote renderer supports every Markdown extension. Apply the target system's restrictions too.

## Failure Handling

1. Do not commit or write remotely while either command fails.
2. For formatting-only Prettier failures, run Prettier on the same explicit files:

   ```bash
   prettier --write "<file-1>" "<file-2>"
   ```

3. For markdownlint errors, edit the Markdown to correct the reported issue. Do not suppress a rule, use blanket disable comments, or change shared configuration unless the user explicitly authorizes that policy change.
4. Re-run **both** required checks after every correction.
5. If a target renderer needs syntax that conflicts with a lint rule, explain the conflict and request an explicit decision rather than silently bypassing validation.

## Delivery Gate

Only proceed with the Git commit or remote write after both commands exit successfully. In the final report, state:

- which Markdown files or temporary body were checked;
- that `markdownlint-cli2` passed;
- that `prettier --check` passed;
- any target-specific validation performed or any known limitation.
