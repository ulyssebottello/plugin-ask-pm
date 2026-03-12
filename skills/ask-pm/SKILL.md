---
name: ask-pm
description: >
  Escalate product decisions to the PM via Linear comments when encountering
  functional ambiguity, UX tradeoffs, or unclear expected behavior.
  Also handles: checking for PM responses, polling while waiting,
  and applying decisions.
  Triggers: ambiguous specs, missing edge cases, UX tradeoffs, scope questions,
  "ask the PM", "check with product", "what did the PM say", "did the PM respond".
argument-hint: "[question or 'check']"
---

# Ask-PM — Product Decision Escalation via Linear

You have access to **Linear MCP tools**. Use them to:
- Create comments on issues (to ask questions)
- Read comments on issues (to check for PM responses)
- Read issue descriptions (to get current spec context)

---

## 1 — WHEN TO ESCALATE

### Escalate when:

- **Functional ambiguity** — Spec uses vague language: "handle errors gracefully",
  "should work well", "nice UX", without concrete behavior definition.
- **UX tradeoffs** — A technical constraint forces a user experience choice.
- **Missing edge cases** — Happy path is specified but not: empty states, errors,
  concurrent edits, permissions, offline, first-time use, bulk actions, etc.
- **Behavioral ambiguity** — Multiple valid interpretations exist.
- **Scope creep** — Implementing feature A properly requires feature B (not in spec).
- **Contradictions** — Two spec requirements conflict with each other.

### Do NOT escalate when:

- Purely technical: library choice, design pattern, performance approach
- Answerable by codebase, docs, CLAUDE.md, or issue description
- Covered by existing conventions or style guides
- Already decided in prior Linear comments on this issue

---

## 2 — HOW TO ASK

Create a Linear **comment** on the issue you're working on.

### Comment format

```markdown
🏷️ **Ask-PM** — Decision needed · @ulysse

**Context:**
[1-2 sentences: what you're implementing and what ambiguity you hit]

**Question:**
[One specific question. Not vague.]

**Options:**
→ **A) [Name]** — [What it does] · _Pro:_ [benefit] · _Con:_ [downside]
→ **B) [Name]** — [What it does] · _Pro:_ [benefit] · _Con:_ [downside]
→ **C) [Name]** — (only if genuinely needed)

**Blocked?** [Yes — what's blocked / No — continuing on X in the meantime]

---
_🤖 Ask-PM · awaiting decision_
```

### Rules

- **Be specific.** Bad: "How should errors work?" Good: "When Stripe webhook fails
  after user sees 'Payment complete': (A) delayed error toast, (B) email, (C) silent retry?"
- **Always propose 2-4 options** with tradeoffs. PM picks, doesn't brainstorm.
- **One decision per comment.** Multiple questions = multiple comments.
- **State what's blocked** so PM can prioritize their response.
- **Always mention `@ulysse`** in the comment so the PM gets notified.

---

## 3 — AFTER ASKING: WHAT TO DO WHILE WAITING

After posting the Ask-PM comment:

### Step 1: Tell the developer and ask what to do

Briefly summarize what you asked, then **ask the developer** using the
AskUserQuestion tool:

> "Question posée au PM sur [ISSUE-ID] à propos de [topic].
> Tu veux que j'attende sa réponse (je poll toutes les 2 min) ou je continue
> sur les tâches non bloquées ?"

### Step 2: Act based on the developer's choice

**If the dev says "attends" / "wait" / "poll":**

Enter the polling loop:

```
1. Run: sleep 120 (wait 2 minutes)
2. Read comments on the issue via Linear MCP (list_comments)
3. Look for any comment posted AFTER your Ask-PM comment
4. If PM response found → go to Section 4 (apply decision)
5. If no response → go back to 1
6. After 15 polls with no response (~30 min) → stop gracefully
   Leave a final message:
   "Still waiting on PM decision about [topic] on [ISSUE-ID].
    Run /ask-pm check to resume when PM responds."
```

**If the dev says "continue" / "passe à autre chose":**

Continue on unblocked tasks. The dev will say "check Linear" or run
`/ask-pm check` later when the PM has responded.

**If running in headless mode (`-p`):**

Skip the question — go straight into the polling loop above. In headless mode
there is no dev to ask, so always poll.

### Step 3: Dev manually checks later

The dev can always say "check if PM responded on ENG-123" or run `/ask-pm check`
and you should immediately read the comments on that issue.

---

## 4 — WHEN THE PM RESPONDS: APPLY DECISION

When you find a PM response to an Ask-PM question:

### Step 1: Announce the decision

Tell the dev: "PM decided on [topic]: [brief summary of decision]."

### Step 2: Apply the decision in code

Implement according to the PM's answer.

### Step 3: Confirm in Linear comments (thread reply)

**Reply in the thread** of the original Ask-PM comment — do NOT create a new
top-level comment. Use `parentId` (the ID of the original 🏷️ Ask-PM comment)
when calling `save_comment`.

```markdown
✅ **Ask-PM** — Decision applied

Applied: [brief description of what was implemented based on PM's decision].
See commit [ref] / PR [ref].
```

**Do NOT update the issue description.** All decisions are traced in the
comment thread.

---

## 5 — SESSION START BEHAVIOR

When beginning work on a Linear issue:

1. **Read the issue description** — this is your source of truth for the spec
2. **Read recent comments** — look for:
   - Pending Ask-PM questions (🏷️ marker) without PM responses
   - PM responses that haven't been applied yet (no ✅ follow-up)
   - Any other PM clarifications in the comments
3. **Brief the developer:**
   - "There's a pending question about [X] — PM hasn't responded yet"
   - "PM decided [X] about [topic] — I'll apply this"
   - Or nothing, if everything is up to date

---

## 6 — QUICK REFERENCE

| Situation | Action |
|-----------|--------|
| Hit an ambiguity while coding | Post Ask-PM comment → continue unblocked work |
| Dev says "ask the PM about X" | Post Ask-PM comment about X |
| Dev says "check Linear" / "did PM respond" | Read comments on the issue |
| `/ask-pm check` | Read comments, report status of all pending decisions |
| `/ask-pm [question]` | Post Ask-PM comment with the given question |
| PM responded | Apply decision → confirm in thread reply |
| Autonomous + blocked + no response after 30min | Stop gracefully, list what's done and what's blocked |
| New session on a ticket | Read description + comments, brief dev on pending/new decisions |
