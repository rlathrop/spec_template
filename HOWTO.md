# How To Use This

This is the actual workflow I use. Not a sanitized version of it. If something here doesn't fit your setup, adapt it.

---

## What You Need Before You Start

**A plan document.** This is whatever you'd normally hand to a developer. It describes what you're building, not how to build it. It can be rough. Ambiguity in the plan is fine because surfacing that ambiguity is part of what the spec process does.

**The relevant source files.** The sections of the codebase the plan touches. You don't need the whole repo, just enough that Claude can see what actually exists, what the function signatures are, what the existing patterns look like. If you're not sure what's relevant, start with the files the plan mentions and let Claude tell you if it needs more.

**The spec template.** That's `template/Agentic_Development_Spec_Template_Generic.md` in this repo. Read through it once before you use it so you understand what each section is for. The inline notes explain which sections to cut for simpler phases.

---

## Step 1: Write or Get a Plan

If you're doing this on a real project you probably already have something. A Jira ticket, a design doc, a README section, a long Slack message explaining what needs to happen. That's your plan. It doesn't need to be formal.

If you're starting from scratch, write a short document that covers what you're building or changing, why, any constraints you already know about, and any decisions you know are open.

Don't overthink it. The spec process will surface what's missing.

---

## Step 2: Give Claude the Plan, the Code, and the Template

Open a Claude session. Give it all three things. Here's roughly the prompt I use:

```
I'm about to implement the following phase of work. I've attached:
1. My plan document
2. The relevant source files
3. My spec template

Please produce a completed spec by doing the following:
- Read the plan and identify every decision that's been deferred or left ambiguous
- Read the source files to find dependencies, contracts, and constraints that aren't in the plan
- For cross-language or platform-sensitive work, flag any known gotchas
- Fill in the spec template with what you find, making actual decisions where the plan left them open
- For the Agent Planning Instructions section, be specific about which files to read and what to confirm before writing anything

Where you're not sure about a decision, make a recommendation and flag it clearly so I can review it.
```

Adjust the prompt for your situation. If it's a security-sensitive phase, say so. If it's a pure refactor, say so. That context shapes what Claude looks for.

---

## Step 3: Review What Comes Back

This is your job. Claude will produce a draft spec. Your role is to:

**Correct mistakes.** If Claude got a function signature wrong or made an assumption about the codebase that isn't true, fix it. This is why you read the source files before handing off.

**Confirm decisions.** Every place the spec resolved an ambiguity from the plan, check that you agree with the resolution. These are the decisions that will constrain the agent. If you disagree with one, change it now.

**Add what's missing.** Claude won't know everything. If there's a constraint specific to your team's conventions, a deployment requirement, a known fragile dependency, add it. The spec is yours, Claude is drafting it.

**Check the planning instructions.** This section should be specific. "Read `src/pkg/scanner/pipeline.go` and confirm the `Config` struct fields" is useful. "Read the relevant files" is not. If the planning instructions are generic, make them specific to this phase.

This review step usually takes me 15 to 20 minutes. Less for simple phases, more for complex ones.

---

## Step 4: Run the Pre-Flight Gate

Before you hand the spec to the implementation agent, verify the pre-flight gate in the spec yourself. These are the conditions that must be true before implementation starts. If any of them aren't met, fix that first.

Don't skip this. Agents that start in a broken environment produce subtly broken output that's hard to diagnose. If the pre-flight gate says "go test ./... passes" and you haven't run it, you're handing the agent a codebase where you don't actually know the starting state. When something breaks later, you won't know if it was the agent or if it was already broken.

---

## Step 5: Give the Spec to the Implementation Agent

Create a branch for the phase. Do not let the agent work on main. If things go sideways — and they will eventually — you want a clean rollback point. A phase branch means the worst case is deleting a branch, not cherry-picking through a contaminated commit history.

Hand the spec to Claude Code, or whatever agent is doing the implementation. Include the spec and the relevant source files. Do not include the holdout scenarios section. Those scenarios are for you, not the agent.

Tell the agent to follow the spec. The planning instructions section tells it what to do before writing anything. Let it produce the written plan first.

---

## Step 6: Review the Agent's Plan

This step is separate from reviewing the final output because it's your highest-leverage checkpoint. The agent produces a written plan before writing any code. Read it.

The plan must include explicit answers to the checkpoint questions at the end of the planning instructions section. If any answer is vague or just paraphrases the spec back at you, push back before implementation starts. Those answers are your signal that the agent actually internalized the constraints rather than pattern-matched on them.

If the plan reveals that the spec got something wrong — a function signature doesn't match, a file doesn't exist where the spec said it would, a dependency has changed — fix the spec before implementation starts. It's cheaper to correct the spec now than to debug a diff later.

This is the last point where course-correcting is cheap. Once the agent starts writing code, the cost of a bad assumption goes up fast.

---

## Step 7: Review the Output

When the agent finishes, run the visible validation scenarios from the spec. These are the ones the agent had access to during implementation.

Then run the holdout scenarios yourself. These are the ones the agent never saw. They're the real test. If something fails here, it means the spec didn't fully constrain the behavior you cared about.

**When holdout scenarios fail**, you have a few options depending on severity. If it's a small gap, patch it by hand and note what the spec should have said. If it's a structural miss — the agent made a design decision that was wrong — it's usually faster to update the spec with the correction, reset the branch, and rerun the phase than to try to patch a bad foundation. Either way, the failure tells you something specific about the spec. Use it.

---

## Step 8: Update the Template

This is the step most people skip. Don't.

If the agent drifted in a way the spec didn't prevent, figure out what section would have caught it. Add it to the template. If a section of the template turned out to be useless for this type of work, note that too.

The template gets better every time you use it. After a few phases it starts to reflect your actual environment, your actual failure modes, your actual constraints. That's when it becomes genuinely powerful.

---

## Tips That Aren't Obvious

**Give Claude the actual source files, not descriptions of them.** Claude reading `pipeline.go` directly will find things you wouldn't think to describe. That's the point.

**The phase type matters.** Tell Claude whether this is a refactor, a new feature, a security-sensitive change, a cross-language integration. That context shapes which parts of the template get the most attention.

**Don't skip the holdout scenarios.** It's tempting to write only visible scenarios because those are the ones that feel immediately useful. The holdout scenarios are what separates a real validation from a rubber stamp. Write at least two or three per phase.

**The persona section is not flavor text.** The agent reads this before anything else. "Read first, plan before writing, fail closed on ambiguity" actually changes how the agent approaches the work. "Security-first, never guess at platform behavior" actually changes how a security-sensitive phase gets implemented. Take it seriously.

**Short phases need short specs.** Phase 4 of IoT Scout was literally two lines added to .gitignore and a build command. The spec for it is mostly a verification checklist. Don't over-template simple work. The inline notes in the template tell you what to cut.

**Always work on a branch.** The agent works on a phase branch, never main. Merge when the phase passes validation. This is not optional. When an agent compounds bad decisions across commits — and it will, eventually — you want a clean main to fall back to, not a contaminated history you have to cherry-pick through.

**When the spec is done, commit it.** The spec lives in the repo alongside the code it governs. When you come back to this codebase in six months you'll want to know what decisions were made and why. The spec is that record.

---

## Common Mistakes

**Writing the spec after implementation.** This happens when people treat the spec as documentation rather than a contract. It's useless as documentation and it defeats the entire point. Write the spec first.

**Leaving TBDs in the spec.** Every TBD is a decision the agent will make alone. If you don't know the answer yet, figure it out before handing off. If it genuinely can't be resolved, it's a blocker, not a TBD.

**Generic planning instructions.** "Read the relevant files before proceeding" does almost nothing. "Read `src/internal/models/types.go` and confirm the JSON tag on the `IsIoT` field before writing any JavaScript that accesses it" does a lot. Be specific.

**Skipping the agent's plan review.** The spec requires the agent to produce a written plan before writing any code. Read that plan. It will sometimes reveal that the agent misunderstood something in the spec. Catch it there, not in the diff.

**Working on main.** Run every phase on a branch. If the agent rewrites tests to make them pass instead of fixing the underlying bug — and this has happened — you want that damage contained, not baked into your mainline history.

**Treating the first spec as the final template.** The template should evolve. The first version will be missing things. That's normal. The value compounds as you refine it.
