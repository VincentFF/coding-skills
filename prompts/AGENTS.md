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

## Preferred local tooling

- Use `rg` for repository text search and `fd` or `rg --files` for file discovery.
  Do not use recursive `grep` or `find` for ad-hoc repository exploration.
- Use `jq` for JSON and `yq` v4 for YAML queries or transformations. Do not
  parse structured data with `grep` or `sed`.
