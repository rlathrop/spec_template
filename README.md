# The Spec Is the Control Plane

> **Jump to:** [Spec Template](template/Agentic_Development_Spec_Template_Generic.md) · [Example Specs](examples/) · [Step-by-Step HOWTO](HOWTO.md)

I transformed a Go CLI security tool into a native desktop application across six implementation phases. Pipeline extraction refactor, OS-native privilege escalation across macOS, Linux, and Windows, a Wails desktop integration, vanilla JS frontend, and a dark theme. A couple days of work. The agent wrote the code. I didn't write most of the specs either — Claude did, from a template I built. But the spec is where the real engineering happens, and that's what this repo is about.

---

## What Went Wrong Before Specs

Before I had this workflow, I was doing what most people do. Give the agent a task, review the diff, course-correct, repeat. It works until it doesn't.

In an early phase of IoT Scout, the agent introduced a change that broke existing tests. Reasonable enough — that happens. What happened next changed how I work. Instead of fixing the underlying issue, the agent rewrote the tests to make them pass. The tests went green. The code was wrong.

I didn't catch it immediately because the signal I was relying on — the test suite — was now lying to me. The agent kept building on top of that broken foundation. By the time I realized what had happened, it had been working on main, compounding bad commits. It took me hours of cherry-picking through the git history to recover a clean state. I nearly started over.

That's not an edge case. That's what happens when the agent has latitude to make decisions you didn't explicitly authorize. The tests were in scope. Modifying them seemed reasonable. The agent wasn't wrong to touch them — it was wrong about *how*, and nothing in its instructions prevented that.

That failure is why the spec template has an explicit constraints section, an invariants section, and a declared file operations table. Every one of those sections exists because something went wrong without it.

---

## What a Spec Actually Does

The thinking you'd normally do during implementation doesn't go away when you use an agent. It just moves to a worse place. The agent hits an ambiguous decision, picks something, and keeps going. You find out three hours later when you're reviewing a diff that's tangled with five other things.

A spec moves that thinking to before the agent touches anything. It's cheaper to fix a decision in a document than in code. When the spec is good, the agent doesn't have to make those calls at all.

There are three places where specs tend to fail.

**Unresolved decisions.** Every place the plan says "figure this out during implementation" is a place where the agent will decide something you didn't authorize. Take Phase 1 of IoT Scout. The plan said "use OS-native elevation, Windows is trickier, may need golang.org/x/sys." That's a landmine. On Windows, `os.Getuid()` always returns -1 — it's useless for detecting elevation. You need the Windows token API. If the spec doesn't say that, the agent writes something that compiles, runs, and silently gives every Windows user wrong behavior. The spec resolved it before a single line was written.

**Missing invariants.** In Phase 0, a pure refactor, the most important invariant was that the new ProgressFunc parameter had to be nilable. Pass nil, don't panic. That's not in any plan document. It's the kind of thing a senior engineer knows to check. Write it down or the agent won't check it.

**Weak planning instructions.** This is the highest-leverage section in any spec. Before the agent writes anything, it must produce a written plan. Read this file. Confirm this function signature. State this contract. Without this, the agent starts implementing and figures out the details as it goes. With it, the agent surfaces mismatches before they become bugs buried in a diff.

---

## How the Specs Get Written

I want to be honest about the workflow, because "Claude wrote the specs" lacks context.

The template didn't exist before IoT Scout. It evolved from it. I had already built the CLI version of the tool and had a basic template forming when I started writing specs for the desktop transformation. The early specs were thinner, and the agent made exactly the decisions I hadn't pinned down. Each phase that went sideways taught me what was missing. By the end of the project the template had grown into what's in this repo — agent persona and operating discipline, invariants, explicit constraints, a failure modes table, holdout validation scenarios the agent never sees, and planning instructions that force a written plan before any file is touched.

The planning instructions section was the last major addition. After IoT Scout, I added a structured 7-step planning process specifically to force the agent's attention span — to make sure it actually read the complete spec before writing anything, rather than skimming the first few sections and starting to code. That section exists because agents pattern-match on partial context if you let them.

Now when I start a phase, I give Claude the plan document, the relevant source files, and the template. I tell it to fill the template in — surface the ambiguities in the plan, read the code to find what the plan didn't say, make the decisions the plan left open, and produce a completed spec in that structure.

Claude brings the code reading and the platform knowledge. The template brings the structure that forces the right questions to get answered. Without the template you get a well-organized summary of the plan. With it you get something an agent can actually execute against.

A solid spec for a mid-complexity phase takes about thirty minutes including review. What it buys is an agent that runs clean for two hours without me. That's not overhead, that's leverage.

---

## How Far It Scales

The dark theme spec tested the low end. I ran a UI audit using a design agent in Claude Code. It came back with a list of problems and recommendations. I pasted that output into a message with the template and typed one line: "Can you turn the following into a UI enhancement spec."

The output was a complete spec that covered the surgical edit model, preserved the Phase 3 security invariants explicitly, listed every CSS class that couldn't be renamed without breaking the JS, and included a full regression gate against the prior phase. I reviewed it, changed nothing significant, and handed it to the implementation agent.

That's only possible because the template had been refined through the harder phases first. A CSS theme pass is low-risk, low-ambiguity work — the kind where a tight template can do almost all of the thinking about what questions need answering. The harder phases are where the spec earns its keep. Phase 1, with three OS-specific privilege escalation mechanisms and a security boundary, is where the spec prevented real damage.

The template keeps compounding. Every phase I run, when the agent drifts in a way I didn't anticipate, I add a section or tighten a constraint.

---

## The Workflow

You have a plan document, the relevant source files, and the spec template. You give all three to Claude and ask it to produce a completed spec. Surface the ambiguities, read the code, make the deferred decisions, fill in the structure. You review what comes back, correct what's wrong, confirm the decisions.

Then you give the spec to the implementation agent. You go do something else for a couple hours. You come back and check if the tests pass.

That's it. It's not a methodology with eight steps. It's using the same tool you're already using for code to write the spec first, then using the spec to constrain the code.

The HOWTO in this repo walks through each step in detail.

---

## A Note on Detail Level

The specs in this repo are detailed. Intentionally. Some people will look at them and say that's too much overhead.

This is how I think about implementation work anyway. The spec doesn't represent extra thinking — it represents the thinking I was going to do regardless, written down before the agent started instead of during or after.

If your natural detail level is lower, your specs will be shorter. The template scales down. But don't mistake brevity for simplicity. Every section that feels like too much exists because something broke without it.

---

## What's In This Repo

The template I used to produce the IoT Scout specs, all six specs from the project, and a step-by-step HOWTO. The specs are real, they worked, and you can see exactly what a completed spec looks like across a refactor, a security-sensitive phase, a cross-language integration, a frontend build, a build-and-verify pass, and a visual enhancement.

Use the template. Adapt it. The sections that don't apply to your work are marked so you know what to cut.

The spec is the control plane. The code is what happens after the engineering is done.

```
template/
  spec-template.md          The generic spec template

examples/
  phase-0-pipeline-extraction.md
  phase-1-privilege-escalation.md
  phase-2-wails-integration.md
  phase-3-frontend.md
  phase-4-build-and-verify.md
  ui-dark-theme.md

HOWTO.md                    The workflow step by step
README.md                   You are here
```
