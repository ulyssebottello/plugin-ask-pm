---
name: ask-pm
description: >
  Escalate product decisions to the PM via Linear comments when encountering
  functional ambiguity, UX tradeoffs, or unclear expected behavior.
  Also handles: checking for PM responses, polling while waiting,
  applying decisions, and updating the issue description as living PRD.
  Triggers: ambiguous specs, missing edge cases, UX tradeoffs, scope questions,
  "ask the PM", "check with product", "what did the PM say", "did the PM respond".
argument-hint: "[question or 'check']"
---

# Ask-PM — Product Decision Escalation via Linear

You have access to **Linear MCP tools**. Use them to:
- Create comments on issues (to ask questions)
- Read comments on issues (to check for PM responses)
- Read issue descriptions (to get current spec context)
- Update issue descriptions (to incorporate decisions into the living PRD)

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
🏷️ **Ask-PM** — Decision needed · @ulyssebottello

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
- **Always mention `@ulyssebottello`** in the comment so the PM gets notified.

---

## 3 — AFTER ASKING: WHAT TO DO WHILE WAITING

This is critical. After posting the question:

### Step 1: Tell the developer what you did

"I've posted a question to the PM on [ISSUE-ID] about [topic]. Here's what I asked: [brief summary]."

### Step 2: Work on unblocked tasks

Continue on anything that doesn't depend on this decision. Tell the dev what you're
working on instead.

### Step 3: Check for the PM response

**If the dev is present (interactive session):**
- Work on unblocked tasks
- The dev will tell you when the PM has responded, or say "check Linear"
- When they do, go to Section 4

**If running autonomously (headless / background agent):**
- After exhausting unblocked work, **poll for the PM's response**
- Use the Linear MCP to read comments on the issue
- Look for any comment posted AFTER your Ask-PM comment that isn't from the bot
- **Poll rhythm:** wait ~2 minutes between checks (use `sleep 120` in bash)
- **Maximum 15 polls** (≈ 30 minutes). After that, stop and leave a summary
  of what was completed and what remains blocked

**Polling loop (autonomous mode):**
```
1. Post Ask-PM comment
2. Work on all unblocked tasks
3. When unblocked work is done:
   a. Read comments on the issue via Linear MCP
   b. If PM response found → go to Section 4
   c. If no response → sleep 120 → go to 3a
   d. After 15 polls with no response → stop gracefully
      Leave a final message to the dev:
      "Completed [X, Y, Z]. Still waiting on PM decision about [topic] 
       on [ISSUE-ID]. Run /ask-pm check to resume when PM responds."
```

### Step 4: Dev manually checks later

The dev can always say "check if PM responded on ENG-123" or run `/ask-pm check`
and you should immediately read the comments on that issue.

---

## 4 — WHEN THE PM RESPONDS: APPLY + UPDATE DESCRIPTION

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
Issue description updated. See commit [ref] / PR [ref].
```

### Step 4: Update the issue description (LIVING PRD)

This is essential. The issue description must always reflect the current,
decided-upon spec. After each PM decision:

1. **Read the current issue description** via Linear MCP
2. **Incorporate the decision** into the relevant section of the description
3. **Update the issue description** via Linear MCP

**How to update the description:**

- Find the relevant section of the description where the decision applies
  and insert a **blockquote** right after the concerned paragraph/section.
- The blockquote traces what was decided: addition, modification, or removal.
- Do NOT just append to the bottom — place it in context.
- If there is no obvious section, add a `## Decisions` section at the end
  with the blockquote(s).

**Blockquote format:**

```markdown
> **🏷️ PM Decision** _(2025-03-12)_ — [Added | Modified | Removed]:
> [Concise description of what was decided and why.]
> _(See thread on issue comments)_
```

**Examples inline in the description:**

```markdown
### Archive behavior

When a project is archived, tasks are soft-deleted after 30 days.

> **🏷️ PM Decision** _(2025-03-12)_ — Modified:
> Tasks remain visible with an "archived" badge instead of being soft-deleted.
> Hidden from default filters but accessible via search.
> _(See thread on issue comments)_
```

```markdown
### Notifications

Users receive email notifications for all activity.

> **🏷️ PM Decision** _(2025-03-12)_ — Removed:
> Email notifications dropped for v1. Only in-app notifications.
> _(See thread on issue comments)_
```

```markdown
### Empty state

> **🏷️ PM Decision** _(2025-03-12)_ — Added:
> Show onboarding checklist on first use instead of blank page.
> _(See thread on issue comments)_
```

**Rules for description updates:**
- Preserve all existing content. Only add or refine — never delete spec content.
- Keep the same tone and format as the rest of the description.
- If the decision contradicts something in the original description, update that
  specific part and place the blockquote right after to explain the change.
- The goal: anyone reading the issue description sees the complete, current spec
  with clear trace of PM decisions — without digging through comment threads.

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
| PM responded | Apply decision → confirm in comments → update issue description |
| Autonomous + blocked + no response after 30min | Stop gracefully, list what's done and what's blocked |
| New session on a ticket | Read description + comments, brief dev on pending/new decisions |
