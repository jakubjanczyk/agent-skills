---
name: writing-tests-autonomously
description: Test-first implementation without a human in the loop - the agent works out the scenarios itself.
disable-model-invocation: true
---

# Writing tests autonomously

Tests are the specification of what the system does and its first user. If a test needs 20 steps to do something, a real user probably does too, so treat awkward tests as feedback on the design.

## How to work

1. **Gather.** Read the request, the features most similar to it and their tests, and the project's existing builders, helpers, and network fakes. Done when you can say which test file each part of the change belongs in, and which existing behavior it most resembles.
2. **Draft the scenarios, top down.** Unless you're prototyping or still figuring things out, start at the highest level you can (for a new page, the page itself). Include the happy paths, edge cases, and error cases that can actually happen and make sense for this feature: ones a real user or caller can run into, where the system should respond in a way worth specifying. Don't add a case just to fill a category. Don't think about the implementation yet. Work from the behavior you were given. Where it doesn't say what should happen, decide and record it as an assumption (step 3).
3. **Challenge the list in rounds.** In each round, go through the list with these lenses:
   - A real user's or caller's journey through the feature.
   - Boundary inputs.
   - Failures that can actually happen: network, validation, permissions.
   - Existing state: empty, populated, another account's data.
   - What the request doesn't say.

   If you can start a subagent, give it the request, your list, and a few facts it can't guess (data shapes, what the system already does, what can't happen), and ask what's missing. Treat its answer as one more lens in the round. Drop suggestions that don't apply and note why in the report.

   For each gap, decide in this order: how similar features behave, then project conventions, then the most conservative option that's easiest to reverse. Record the decision as an assumption with a one-line reason. If the requirements contradict each other, pick the more conservative reading and put the conflict first in the report. Remove scenarios that can't happen or only fill a category.

   Done when a round changes nothing, or after 2 rounds.

4. **Write the list down.** Post the scenarios, grouped by the file each goes in, and the assumptions in your message, then carry on.

   ```
   tests/api/delete-tag.test
     - deletes the tag
     - removes a deleted tag from its subscribers
     - returns not found for a tag from another account

   tests/pages/tags-page.test
     - removes a tag from the list after deleting it
     - keeps the tag when deleting fails
     - shows an error when deleting fails
   ```

5. **Write the tests before the implementation.** Required.
   - Reuse the project's existing builders, helpers, and network fakes. When one you need lives in another test file, move it somewhere shared instead of copying it.
   - Preferably one scenario at a time: write a test, see it go red, write enough code to turn it green, refactor, then take the next one. Writing tests for all known scenarios first is also fine.
   - Red means the test fails on its assertion, not on an error like a missing class or route. If it errors, add the smallest stub (an empty route, action, or component) and run it again until the assertion is what fails.
   - Make stubs return something no test expects. A stub that returns an empty result can make an "ignores X" or "returns nothing" test pass without any code.
   - A test that checks something is absent ("doesn't show X when Y") passes before the code exists. That's expected. Its red check is the branch flip in step 6.
   - Refactor only when green: clean up both the code and the tests, and run the tests after each change. Delete tests that newer tests have made redundant: a test is redundant when it can't fail without another test failing for the same reason.
   - If a test ends up written after its code, that's still far better than no test: break the code, see the test go red, then restore it.
   - Everything in one file is fine at first. When it gets too complex, split it up, but keep testing at the top level.
6. **Check the branches that could be passing by accident.** Once everything is green, flip or remove a branch, confirm a test goes red, then restore it. Do this for:
   - tests that were green before their code existed (usually absence checks like "omits X when Y"),
   - tests written after their code,
   - conditions where a test could pass for an unrelated reason (a stub returning the expected value, data that dedupes on insert, a default that matches the expectation).

   Skip branches whose test already went red on that exact behavior in step 5. If no test goes red, either a scenario is missing (add it through step 5) or the branch isn't needed (delete it). If the branch is needed but can't change any result (for example, a filter kept for query performance), keep it and say so in the report.
7. **Revisit the list once.** With the code written, run one more round of step 3. Add anything new through step 5.
8. **Go through "Before you finish"**, then write the report.

**Bugs: red first, always.** Reproduce the issue with a test that fails the same way as the report (Sentry, a ticket) before touching the fix. A fix found by reading code might not fix the real problem.

## Report

Your final message must have these six sections, in this order, each with a heading. If a section has nothing to say, write "None" rather than leaving it out.

1. Assumptions, each with its reason.
2. The scenarios, grouped by file.
3. Scenarios considered and dropped, and tests deleted as redundant with the test that covers each, one line each.
4. Tests written after their code, and why.
5. Test speed: the new tests' running time compared with neighbouring test files, and any variations moved one level lower.
6. "Before you finish" items you didn't do, and why.

## Test behavior, not implementation

This is the most important rule. Check what the system does, not how it does it.

- ✅ Clicking the button starts a download. ❌ Method `a` was called.
- ✅ The endpoint returns this response for this input. ❌ The repository was used.

If you refactor everything and the behavior stays the same, the tests stay the same too. Tests that break on every refactor are why people stop writing tests. This applies to frontend and backend alike.

What to assert:

- **The outcome that matters.** An endpoint that deletes a record, with a test that only checks for a 204, has 100% coverage but never checks that the record is gone. Coverage is not the goal.
- **Not details that don't matter.** Whether an element has a certain class usually doesn't.
- **Expected values written out independently.** Don't compare a value with itself (the same variable or mock result). Write the expected value again as primitives, so a check on identity instead of value gets caught.
- **The current behavior, not the history.** If a button gets removed, don't add a test that it's gone. Only check that something is absent when the absence is itself a behavior someone asked for, for example "viewers without permission don't see Delete".
- **No snapshots.** They lock in markup rather than behavior.

When a test goes red, it's the source of truth: fix the implementation, never weaken the test. Two exceptions: if the behavior itself changed, update the test. If a refactor broke it but the behavior didn't change, the test was coupled to implementation details, so fix that coupling and don't bend the code to fit it.

Tests set patterns that get copied. Don't treat an existing bad test as the standard. If there's a clearly better approach, write new tests that way.

## Pick the right level: a trophy, not a pyramid

The test pyramid (mostly unit tests) is the wrong model. Use the testing trophy: some unit tests, some e2e tests, mostly integration tests. An integration test is also a unit test. It just tests a bigger unit.

- Pick the unit by what works together in production. A design-system button is fine to test alone. A page that uses it renders the real button.
- Always have at least one integration test, ideally a few.
- Add lower-level tests to cover more cases cheaply, never to test implementation details. They fit when a piece has something of its own to test: a reusable component that can grow independently of where it was first used, or an algorithm with many cases that depend only on its input.
- Start with every scenario at the top level. Then check how long the new tests take to run. If they're noticeably slower than similar tests in the suite, move specific variations one level lower, where they run faster: extra input cases, or branches of an algorithm. Keep each scenario covered at the top level by at least one test, and only move the extra cases down.

Example: an endpoint reads from the DB, processes the data, and returns it. Test the endpoint through the whole stack. If the processing has many conditions that depend only on the data, also test it on its own.

## Don't mock

The default is no mocks. Don't mock components, interfaces, hooks, or your own modules. When everything is mocked, you only know the pieces work in a bubble that never exists in production.

Use real dependencies wherever practical: a connected test database, Redis, or an in-memory equivalent. These aren't mocks.

Replace something only at the boundary of your system:

- **The network.** Fake it at the network level, never by stubbing `fetch` or the app's own client, so requests and response formats behave like the real thing.
- **DOM conditions that are hard to create.**
- **Something very heavy** that slows the suite a lot, like a tooltip library that made every test much slower.

Every replacement costs maintenance, so think twice even then. Inside a small, self-contained unit, a mock is fine for a corner case that real dependencies can't produce.

## Write tests that read like a story

- **Given / when / then.** Setup, action, expected result, in that order.
- **One test, one behavior**, so it has a single reason to fail. A behavior is one action and its outcome. Showing existing data and changing it are two behaviors, so they're two tests. If a test name needs "and" to join two outcomes, split it. Several assertions are fine when they all describe that one outcome.
- **No logic in a test body**: no `if`, loops, or try/catch. Branching hides which case actually ran. For variations of one behavior, use a parametrized test instead of copy-pasted near-duplicates.
- **Hide the mechanics in helpers, keep the meaning in the test.** How data is set up, how an element is found, how a request is faked: that goes into helpers, so when it changes you fix one place. What the scenario is about stays in the test: the inputs that matter, the action, and the expected values. A reader should understand the scenario without opening a helper.
- **Use builders for test data.** A builder creates a valid object with sensible defaults, and each test overrides only the fields its scenario cares about. The overrides show the reader what matters.
- **In UI tests, find elements the way a user sees them**: by role, label, or visible text. Avoid IDs, class names, and test IDs unless there's genuinely no other way.
- **Tests are isolated.** They never depend on each other or on order. Reset shared state between tests: database rows, faked network responses, globals.

### Example

Three scenarios for a tags page. The page renders with its real components, and only the network is faked. The library calls are illustrative; use whatever the project has.

```ts
it("removes a tag from the list after deleting it", async () => {
  givenTags([aTag({ name: "Newsletter" }), aTag({ name: "Customers" })]);

  renderTagsPage();
  await deleteTag("Newsletter");

  await expectVisibleTags(["Customers"]);
});

it("keeps the tag when deleting fails", async () => {
  givenTags([aTag({ name: "Newsletter" })]);
  givenTagDeletionFails();

  renderTagsPage();
  await deleteTag("Newsletter");

  await expectVisibleTags(["Newsletter"]);
});

it("shows an error when deleting fails", async () => {
  givenTags([aTag({ name: "Newsletter" })]);
  givenTagDeletionFails();

  renderTagsPage();
  await deleteTag("Newsletter");

  expect(await errorMessage()).toBe("Couldn't delete the tag. Try again.");
});

// Builder: a valid tag by default; tests override only what their scenario cares about.
function aTag(overrides: Partial<Tag> = {}): Tag {
  return { id: nextId(), name: "Tag", subscriberCount: 0, ...overrides };
}

// Setup: fakes the network at its boundary, not the app's own data-fetching code.
function givenTags(tags: Tag[]) {
  fakeServer.respondTo("GET /tags", { status: 200, body: tags });
  fakeServer.respondTo("DELETE /tags/:id", { status: 204 });
}

function givenTagDeletionFails() {
  fakeServer.respondTo("DELETE /tags/:id", { status: 500 });
}

// Actions and queries: find elements the way a user would, by role and name.
async function deleteTag(name: string) {
  await click(await findByRole("button", { name: `Delete ${name}` }));
  await click(await findByRole("button", { name: "Confirm" }));
}

async function errorMessage() {
  return (await findByRole("alert")).textContent;
}

async function expectVisibleTags(names: string[]) {
  await waitFor(() => {
    const links = getAllByRole("link");
    expect(links.map((link) => link.textContent)).toEqual(names);
  });
}
```

The tests say what someone does and sees. The helpers hold how. If the delete button moves into a menu, only `deleteTag` changes. If deleting leaves the tag on the page, the first test goes red on exactly that. If a failed request removes the tag anyway, the second one does. If it shows no error, the third one does.

## Before you finish

Go through every item. List any item you didn't do in the report, with why.

- [ ] The scenario list went through challenge rounds until one changed nothing, or 2 rounds passed. Every scenario on it has a test.
- [ ] Every assumption is in the report with its reason.
- [ ] Every branch from step 6 makes a test go red when flipped or removed, or is noted in the report as needed but unable to change a result.
- [ ] Every new test was seen red for the right reason: wrong behavior, not an import or syntax error. Tests written before their code went red first. A test written after its code went red when that code was broken, then green once it was restored. A new test that was green before its code existed was investigated: either the behavior already existed, or the test doesn't check what you think. After changing a test or its helpers, the red check was redone.
- [ ] Existing tests you rely on to cover moved or changed code exist, and go red when you break that code.
- [ ] Every test can fail for a reason no other test fails for. Tests that duplicate another, or that newer tests made redundant, were deleted. Every deleted test is called out, with the test that covers its behavior now. Overlap across levels is fine when the lower test covers variations the top one doesn't.
- [ ] Every new and changed test passed on repeated runs. Flakiness, usually from timers, promises, or async code, was fixed at its cause, not hidden.
- [ ] The new tests' running time was compared with neighbouring test files. Slow variations were moved one level lower, with each scenario still covered at the top level.
- [ ] All tests covering the changed code were run, not only the new ones.
- [ ] Any behavior change made to make a test easier to write is called out, not slipped in.
