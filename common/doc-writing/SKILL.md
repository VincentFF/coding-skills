---
name: doc-writing
description: Load only before creating or revising a persistent project Markdown document saved for human readers, such as a design document, report, review, README, or runbook. Do not load for agent Plan mode, chat task plans, implementation reasoning, or internal notes, even when they use Markdown.
---

# Doc Writing

## When to load

Load this skill only when creating or revising a project document that is written to disk and intended for people to read.

Examples: design documents, proposals, reports, reviews, READMEs, runbooks, and user-requested plans saved in the project.

Do not load this skill for an agent's task-planning process, including Plan mode output, chat responses, implementation reasoning, checklists, or temporary internal notes. This applies even if that content is formatted as Markdown or is called a plan.

Write technical documents that are concise, direct, and readable. Follow these rules in order of priority.

## 1. Structure

- Skeleton first: section headings define structure; each section serves one purpose.
- Body states conclusions and rules; details (full configs, commands, expressions) go to appendices.
- Design documents (plans, proposals, schemes): section order is Scope → Taxonomy → Design → General rules → Appendices. Other document types use their own skeleton (e.g. README: purpose → install → usage; investigation report: conclusion → evidence → timeline).

## 2. Language style

- One sentence per idea; short sentences, one information point each.
- Chinese for prose, English for technical terms.
- Tables over paragraphs: if it can be a table, don't write prose. Headers are noun phrases — no "Notes" or "Remarks".
- No slogans or metaphors. Don't write "alerts are phone calls, dashboards are checkups."
- No disclaimers. Don't write "needs verification" or "TBD" — status goes in a table column, not an annotation block.

## 3. Meta-information ban

- No header disclaimer ("This document is effective as of...").
- No version footer ("Document version v1.0").
- No structural self-description ("This document is organized by component layer...").
- No "This document does not redefine XX" — either reference it or write it. Never say you're not writing it.
- No summary statistics line ("Total: 85 rules, 39 replaced...") — readers can count.

## 4. Taste rules

- Each piece of information appears exactly once. Duplicates stay at the most actionable location.
- In appendices, never write "same as above" — write the full reason in every row.
- No label-style reasons ("not covered by design", "XX's responsibility"). State facts directly: "metric xxx does not exist", "no such object in cluster".
- Appendices that record decision history (comparison tables, migration ledgers) preserve all historical entries; never delete established fact rows, but update their reason wording when facts change.
- Inconsistency is the worst readability bug. Totals, category names, and status values must match throughout the document.

## 5. Table rules

- Column names are concise: "Reason" not "Reason for no replacement".
- Status columns contain only persistent values: "Enabled"/"Disabled", not transient states ("Ok"/"Alarm").
- Every row is self-contained: no dependency on the previous row.
- Long text (commands, URLs, expressions) goes in appendix code blocks, not table cells.

## 6. Revising existing documents

- When revising a document with an established structure and style, follow its existing conventions. Do not restructure it to match these rules.
- Apply these rules in full when creating a new document, or when the user asks to rewrite, polish, or optimize an existing one.
