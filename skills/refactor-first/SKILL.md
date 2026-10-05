---
name: refactor-first
description: Before writing code for a task, look for refactors that would make it easier or clean up the code around it, and propose them first.
disable-model-invocation: true
---

"Make the change easy, then make the easy change." Before writing code for the task, look for two kinds of refactoring:

- **Preparatory refactoring**: reshape the code so the task fits in cleanly.
- **Boy Scout rule**: leave the code around the change better than you found it.

Existing code is evidence, not precedent: follow its patterns only after judging them worth following.

1. **Map the touch points.** Investigate where the change lands, or use the investigation or plan already done in this session. List every file, component, and method the task will change or extend, plus their callers, siblings, and specs.

2. **Judge each touch point.** Ask:
   - Would the task add another copy of something that already exists?
   - Does an existing component, helper, or concern almost fit, needing only extraction or generalisation?
   - Would the task grow a method, conditional, or component that is already too big?
   - Do misleading names, dead branches, or leftover flags make the area hard to follow?
   - Is the abstraction wrong, so the task would have to work around it?

   Done when every touch point from step 1 has a verdict: candidate, or fine as is.

3. **Filter.** Keep only candidates that make this task simpler or that improve code at its touch points. Drop anything speculative ("might be useful later"). If nothing survives, say so in a sentence and go to step 5.

4. **Propose, then stop.** For each candidate give:
   - what changes, with file links
   - why: how it makes this task easier, or what it improves around it
   - size (small / medium / large) and whether test coverage is good enough to do it safely
   - placement: separate PR before the task (stack the task branch on it), its own commits at the start of the task's PR, or skip

   Recommend which to do. Don't touch code until the user decides.

5. **Execute.** Do the agreed refactors first, then build the task on top.
