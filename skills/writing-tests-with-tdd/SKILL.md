---
name: writing-tests-with-tdd
description: Strict test-driven development - red, green, refactor, one test at a time.
disable-model-invocation: true
---

# Writing tests with TDD

This is test-driven development. You build the change in a cycle of red → green → refactor, one test at a time:

- No production code without a failing test that asks for it.
- Never more than one failing test at a time.
- Every green is followed by a refactor step.

Tests are the specification of what the system does and its first user. If a test needs 20 steps to do something, a real user probably does too, so treat awkward tests as feedback on the design.

## How to work

1. **Write the test list, top down.** Start at the highest level you can (for a new page, the page itself). This list is your first output on the task, before any code. Include the happy paths, edge cases, and error cases that can actually happen and make sense for this feature: ones a real user or caller can run into, where the system should respond in a way worth specifying. Don't add a case just to fill a category. Don't think about the implementation yet. Work from the behavior you were given. Where it doesn't say what should happen, ask instead of deciding it yourself. Order the list: the first test is the simplest happy path at the top level, a thin slice through the real stack. After that, each next test should force the smallest next change to the code.
2. **Stop and get the test list approved.** Required unless the change is small and targeted.
   - Show the test names you'll add, grouped by the file each goes in. Put the full list in the message where you ask for approval, not only in an earlier message, a plan, or a file.
   - Wait for the user to approve, cut, or add to it. Write no test or code before they reply, then work from the approved list.
   - The task request is not approval, even when it says "implement it" or "go ahead". Only a reply to the list counts, or the user explicitly saying to skip it.
   - Small and targeted means one behavior the user already described, or a follow-up to work agreed in this session. If you're not sure, it isn't: ask.

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

3. **Run the cycle once per test**, in list order (see "The cycle"). Reuse the project's existing builders, helpers, and network fakes. When one you need lives in another test file, move it somewhere shared instead of copying it. Everything in one test file is fine at first. When it gets too complex, split it up, but keep testing at the top level.
4. **Keep the test list up to date.** After each cycle, cross off the test. Anything you discover goes on the list, not straight into code, including a test added only to force the general version of a hardcoded value. Call out tests added beyond the approved list in your final message. If one needs behavior the user never described, ask. When a test from the list passes straight away, find out why before keeping it. If another test already checks the same behavior, delete it and cross it off. If it checks something no other test does, keep it and prove it by breaking the code. A test passing straight away usually means an earlier green did more than its test asked for, so keep greens smaller.
5. **After the last cycle**, compare the new tests' running time with neighbouring test files. If they're noticeably slower, move specific variations one level lower as a final refactor, keeping each scenario covered at the top level. Then go through "Before you finish".

**Bugs: red first, always.** The first cycle reproduces the issue with a test that fails the same way as the report (Sentry, a ticket), before touching the fix. A fix found by reading code might not fix the real problem.

**Untested code you're about to change.** First write a test of what it does today. It goes green straight away, so break the code, see the test go red, and restore it. Then change the behavior through the cycle.

## The cycle

### 🔴 Red

Write exactly one test and run it. It's red when it fails on its assertion, not on an error like a missing class or route. If it errors, add the smallest stub (an empty route, action, or component) and run it again until the assertion is what fails. If it passes, either the behavior already exists or the test doesn't check what you think. Find out which before moving on.

### 🟢 Green

Write the least code that makes the test pass. A hardcoded value is fine: the next test forces the general version. When the general version is as simple as the fake, write it directly. Duplication and ugly code are fine too; cleaning them up is what refactor is for. Least means no behavior the tests don't ask for, not fewer layers: a top-level test can still need a route, a controller, and a model. Green when the whole test file passes, not only the new test.

### ♻️ Refactor

Never skipped. Refactoring changes structure, never behavior: generalizing a hardcoded value is behavior, so a test has to force it. Clean up the implementation: duplication, names, structure the last few cycles made awkward. Then tidy the tests if they need it, and delete tests that newer tests have made redundant. Run the tests after each change. Done when you've said what you changed, or that there's nothing to refactor yet.

### Cycle log

After each cycle, write one short line. Only add detail when something was notable: a red that wasn't an assertion failure, a test that passed straight away, a green that changed existing tests, or a refactor that changed something. For the tags page, where renaming a tag already shows an error on failure:

```
✅ removes a tag from the list after deleting it: filtered the deleted tag out after the request
✅ keeps the tag when deleting fails: filter the list only when the response is ok
✅ shows an error when deleting fails: copied the failure alert from the rename handler
   ♻️ extracted showRequestError(message), used by the rename and delete handlers
```

### Ways the cycle breaks

- Writing a second test before the first is green.
- Writing code no test asks for.
- Refactoring while red.
- Changing behavior during refactor.
- Skipping the refactor.
- Changing the test to make it pass.

## When to step out of the cycle

In very complicated cases, getting a test to fail first can cost far more than the change itself, for example when you don't yet know how to drive the code from a test at all. If a couple of honest attempts haven't produced a meaningful red, build the solution first, then write the test: break the code, see the test go red, and restore it. A working solution with a test written after it is better than hours spent on the first test. Go back to the cycle for the next test. In your final message, call out each test written this way and why.

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

Tests set patterns that get copied. Following the existing tests in a file is fine when it's needed, for example to reuse its setup or stay consistent. Don't spread a clearly worse pattern into new files, and you don't need to rewrite existing tests you only add to.

## Pick the right level: a trophy, not a pyramid

The test pyramid (mostly unit tests) is the wrong model. Use the testing trophy: some unit tests, some e2e tests, mostly integration tests. An integration test is also a unit test. It just tests a bigger unit.

- Pick the unit by what works together in production. A design-system button is fine to test alone. A page that uses it renders the real button.
- Always have at least one integration test, ideally a few.
- Add lower-level tests to cover more cases cheaply, never to test implementation details. They fit when a piece has something of its own to test: a reusable component that can grow independently of where it was first used, or an algorithm with many cases that depend only on its input.
- Start with every scenario at the top level. Only move extra variations lower when the tests get slow (step 5 of "How to work").

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
- **No logic in a test body**: no `if`, loops, or try/catch. Branching hides which case actually ran. For variations of one behavior, use a parametrized test instead of copy-pasted near-duplicates. Write the first variation as a plain test and get it green, then turn it into a parametrized test with the remaining variations. That's the next red for any variation the code doesn't handle yet. Never start with several variations failing at once.
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

Go through every item. In your final message, list any item you didn't do and why.

- [ ] The user approved the test list, unless the change was small and targeted or they said to skip it. Every approved scenario has a test. Any test added beyond the approved list is called out.
- [ ] Every cycle is in the cycle log, with notable reds, greens and refactors called out.
- [ ] Every new test was red for the right reason before its code existed: wrong behavior, not an import or syntax error. A test written after its code (stepping out of the cycle) went red when that code was broken, then green once it was restored, and is called out. After changing a test or its helpers, the red check was redone.
- [ ] Untested code that was changed got a test of its current behavior first.
- [ ] Existing tests you rely on to cover moved or changed code exist, and go red when you break that code.
- [ ] Every test can fail for a reason no other test fails for. Tests that duplicate another, or that newer tests made redundant, were deleted. Every deleted test is called out, with the test that covers its behavior now. Overlap across levels is fine when the lower test covers variations the top one doesn't.
- [ ] Every new and changed test passed on repeated runs. Flakiness, usually from timers, promises, or async code, was fixed at its cause, not hidden.
- [ ] The new tests' running time was compared with neighbouring test files. Slow variations were moved one level lower, with each scenario still covered at the top level.
- [ ] All tests covering the changed code were run, not only the new ones.
- [ ] Any behavior change made to make a test easier to write is called out, not slipped in.
