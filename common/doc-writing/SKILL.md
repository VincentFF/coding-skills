---
name: doc-writing
description: Load before creating or revising an article or document saved to a file for human readers, such as an explainer, design document, report, review, README, or runbook. Do not load for agent Plan mode, chat task plans, implementation reasoning, or internal notes, even when they use Markdown.
---

# Doc Writing

## When to load

Use for articles and documents written to disk for people to read, including user-requested plans saved in the project. Do not use for an agent's task-planning process, ordinary chat responses, or temporary internal notes.

The goal is for the intended reader to understand the point, its basis, and what to do with it without unnecessary rereading. Prioritize factual accuracy, reader comprehension, then brevity and formatting. Do not shorten away necessary context or uncertainty.

## 1. Start with the reader's questions

- Before drafting, identify the intended reader, what they already know, and what they need to understand or do. Use the request and available context; ask only when a missing answer would materially change the document.
- Choose the questions the document must answer, then organize around them. A design might explain the problem, proposed change, rationale, limitations, and validation. These are prompts, not mandatory headings.
- Put the main answer early. Place its explanation, evidence, examples, and relevant conditions nearby so the reader can follow the reasoning.
- Each section should answer a distinct question. Do not create sections just to fill a template or add a summary to every section.
- Choose prose for explanation, ordered lists for procedures, and tables for comparisons with shared fields. Use code blocks for commands, configurations, logs, and exact errors. Move reference detail to an appendix only when it interrupts the main explanation.

## 2. Make claims concrete

- When explaining a change, identify who or what acts, what changes, under which relevant conditions, and with what result. Include what the reader needs; do not force every sentence into this pattern.
- Check abstract phrases such as "统一机制", "能力建设", and "协同优化": do they name a defined concept, or substitute for an explanation? Replace the latter with observable actions or effects.
- Give the factual reason for a decision. "Not covered by design" and "another team's responsibility" are labels, not explanations of why the choice is appropriate.
- Use an example when it makes an unfamiliar rule or relationship easier to understand. Show the behavior or difference, not just the same claim in other words. Do not invent details, measurements, or guarantees to make prose sound specific.

Example — assuming the configuration change is established:

- Vague: "通过统一配置管理，提升配置维护效率。"
- Concrete: "三个服务改为读取同一份配置，修改超时时间时不再需要分别改三处。"

## 3. Keep the reasoning readable

- Give each sentence one main point, keeping the conditions, causes, or contrasts needed to understand it. Split overloaded sentences, not connected reasoning merely to make sentences short.
- Make the connection between problem, evidence, and conclusion explicit. Do not jump from a symptom to a recommendation without explaining how the recommendation addresses it.
- Prefer familiar words and direct verbs over stacked nouns, slogans, or inflated phrasing. An analogy may help explain a concept, but must not replace the actual explanation or imply false equivalence.
- Follow the user's language; default to Chinese prose when unspecified. Use technical names familiar to the audience, preserve exact identifiers, and explain unfamiliar terms on first use. Do not switch to English merely because a term is technical.
- Use specific headings. Replace ambiguous references such as "相关能力" or "上述机制" with their referents when the context is unclear.
- Distinguish facts, inferences, and unknowns. State an uncertainty and its consequence near the affected claim, rather than adding a generic disclaimer or pretending certainty: "尚未进行并发测试，因此不能判断峰值吞吐量。"

Example — preserve the causal relationship rather than fragmenting it:

- Fragmented: "请求会重试。最多重试三次。这样可以提高稳定性。重复提交可能发生。"
- Connected: "请求失败后最多重试三次，因此接收方需要识别重复请求，避免同一订单被创建多次。"

## 4. Remove repetition without losing context

- Explain a point fully where it is most useful. Elsewhere, use a brief reference or a short recap only when it helps the reader navigate or act.
- Consecutive paragraphs and list items may share clearly stated context. Include necessary conditions in items likely to be quoted or executed independently; do not repeat the whole background in every item.
- For recurring appendix reasons, state the shared rule once and reference it precisely. Do not use an ambiguous "same as above" or copy the full explanation into every entry.
- Remove semantic duplicates, not just repeated words. A paragraph that restates the previous conclusion with different adjectives still repeats it.
- Omit ceremonial introductions, structural self-description, and closing slogans. Keep scope boundaries, dates, versions, counts, and summaries only when they help readers interpret the document, make a decision, or find information.

## 5. Revise existing documents with care

- For targeted edits, preserve established structure and terminology outside the affected area. Do not reorganize the document just to match a preferred outline.
- For an explicit rewrite or readability improvement, change structure where it helps the reader; preserve meaning, constraints, and relevant evidence.
- Preserve decision history and historical facts. If circumstances change, record the new fact or superseding decision rather than silently rewriting what was true before.
- Keep names, categories, status values, and totals consistent throughout the document.

## 6. Edit the draft before delivering

After drafting, make a separate editing pass. Perform these checks on the text; do not append the checklist to the document unless requested.

1. **Purpose:** Can the intended reader find the main answer and understand why it matters? Remove sections that answer no relevant question.
2. **Empty prose:** Does each sentence add a fact, explanation, reason, or useful instruction? Delete sentences whose removal loses nothing the reader needs, including generic praise and transitions that merely announce the next section.
3. **Repetition:** Do the introduction, body, and conclusion repeat the same point without adding value? Keep the full explanation in one place and only useful recaps elsewhere.
4. **Clarity:** Are actors, actions, conditions, and references clear where needed? Replace vague abstractions and explain unfamiliar terms.
5. **Reasoning:** Are necessary causal links and qualifications present? Use an example if it resolves a real comprehension gap; remove examples that only paraphrase the rule.
6. **Accuracy:** Are claims supported, unknowns explicit, and terminology and numbers consistent? Do not turn an assumption into a fact while polishing.
7. **Read-through:** Read consecutive paragraphs together. Repair abrupt jumps and choppy fragments; remove unnecessary headings or list breaks. Stop when further shortening would remove needed meaning.
