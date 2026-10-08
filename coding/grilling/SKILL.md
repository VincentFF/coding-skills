---
name: grilling
description: Grill the user relentlessly about a plan, decision, or idea. Use when the user wants to stress-test their thinking, or uses any 'grill' trigger phrases.
license: MIT
---

Interview the user until the consequential decisions for the current objective are clear. Keep the scope bounded: do not reopen settled choices or explore hypothetical future branches merely to exhaust a tree. Map unresolved choices as a **design tree**: every decision branches into the decisions that hang off it.

Work the tree in **rounds**. The **frontier** is every decision whose prerequisites are already settled: the questions you can ask _now_ without guessing at answers you haven't heard yet. Ask the whole frontier in one round: number each question and give your recommended answer. Then wait for the user's answers before the next round.

Format a round like so:

```
❓ **Q1** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>

---

❓ **Q2** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>
```

Word each question so "yes" accepts your recommended answer. Each round the user answers reshapes the tree: settled decisions push the frontier outward and unblock questions that depended on them. Recompute the frontier and ask the next round. A question whose answer depends on another question still open in this round belongs to a _later_ round, not this one.

Finding _facts_ is your job, never the user's. Look up environment facts directly by default. Delegate a bounded investigation only when the user explicitly requested delegation or an applicable instruction authorizes it; complexity alone is not authorization. A running investigation is an unsettled prerequisite: only dependent questions wait, while independent questions can proceed. The _decisions_ are the user's: present them and wait.

The session is done when decisions necessary for the current objective are settled and the user confirms shared understanding. Discussion does not authorize file writes or implementation. Only an explicit documentation or execution request permits the corresponding next stage; do not automatically create ADRs or update long-term contracts.

Adaptation and attribution: [sources](../agent-workflow/references/sources.md).
