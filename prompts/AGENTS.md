## Communication style

- Keep answers short and direct. Lead with the answer, decision, or result;
  do not use filler, excessive politeness, marketing language, or restate the
  user's request.
- Use clear, simple language. Define unavoidable jargon before using it.
- When a user seeks an answer or discussion rather than explicitly requesting
  a change, respond directly. Do not write or modify files, or run
  implementation commands.
- For non-trivial designs and problems, explain: problem, short example or
  trace, then solution. Prefer concrete behavior, and separate necessary work
  from optional complexity.
- Include only evidence-supported caveats, assumptions, risks, alternatives,
  and follow-up work that materially affect the answer or decision.
- When feedback changes the next action, state the resulting decision before
  acting.
- Apply this style to documentation as well: include the context, rationale,
  and examples the reader needs to act; omit background the reader does not
  need.

## Tool selection

- Default to the built-in file tools, when available (`read`, `write`,
  `edit`, `grep`, `find`, `ls` or their equivalents) for reading, writing,
  and searching files. Do not use shell commands like `cat`, `grep`, or
  `find` for tasks the built-in tools can handle.
- Use bash only for what the built-in tools cannot do: running builds,
  tests, git, pipelines, and data processing. In shell commands:
  - Search text with `rg`; locate files with `fd` (named `fdfind` on
    Debian/Ubuntu) or `rg --files`. Avoid recursive `grep -r` and `find`.
  - Process JSON with `jq` and YAML with `yq` v4 instead of parsing with
    `grep`/`sed`. If a preferred tool is unavailable, fall back and note it.
