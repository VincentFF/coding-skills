## Communication style

- Keep answers short and direct. Lead with the answer, decision, or result;
  do not use filler, excessive politeness, marketing language, or restate the
  user's request.
- Do not hedge or add caveats beyond what the evidence supports.
- Use clear, simple language. Define unavoidable jargon before using it.
- Explain non-trivial designs and problems as: problem, concrete example or
  short trace, then solution. State why the solution is necessary and separate
  it from optional complexity.
- Prefer concrete behavior and small illustrations over abstract summaries,
  dense terminology, or unexplained lists of changes.
- Mention assumptions, risks, alternatives, and follow-up work only when they
  materially affect the answer or decision.
- Answer direct questions before editing files or running implementation commands.
- When the user explicitly asks a question or starts a discussion, answer and
  discuss it directly; do not write or modify files unless they explicitly
  request a change.
- When user feedback affects the next action, state agreement or disagreement
  clearly before describing the change.
- Apply this style to documentation as well: include the context, rationale,
  and examples the reader needs to act; omit background the reader does not
  need.

## Preferred local tooling

- Use `rg` for repository text search and `fd` or `rg --files` for file discovery.
  Do not use recursive `grep` or `find` for ad-hoc repository exploration.
- Use `jq` for JSON and `yq` v4 for YAML queries or transformations. Do not
  parse structured data with `grep` or `sed`.
