+++
title = "My AI-Assisted Frontend TDD Workflow"
date = 2026-04-07
draft = false
tags = ["TDD", "AI", "Frontend", "Testing", "MSW"]
+++

## Background

AI writes code fast, but speed comes with two notable risks. First, the scope of changes is unpredictable — you ask it to modify one module, and it sometimes changes adjacent code as well, causing cascading issues elsewhere. Second, the output isn't necessarily correct — code that runs doesn't mean the logic is right, and AI can confidently produce something that looks fine but is actually wrong.

<!--more-->

If these problems aren't caught during development and make it all the way to QA, the cost of tracking them down is far higher than catching them at the coding stage. So the question becomes: **is there a way to automatically catch these issues during coding, rather than relying on manual verification?**

Our answer is TDD. I practiced an AI + TDD workflow on an internal tool platform project called Skills Hub. This article documents the core approach, technical choices, and real cases.

## TDD Workflow: Red → Green → Refactor

The workflow follows the Red-Green-Refactor cycle, driven by AI assistance.

**Step one, Red phase — write tests.** After reviewing the requirements and API documentation, have AI think through what needs to be verified, then generate the corresponding test cases. At this point, all tests must be failing because no implementation code exists yet. This red state is expected — it confirms that the tests themselves are meaningful, not green-by-default decorations.

**Step two, Green phase — implement code.** The core principle is to write the minimum code needed to make tests pass, without over-engineering. Tests are the acceptance criteria — passing them is what counts. This step can be delegated to AI — the tests are its constraints, no additional natural language instructions needed.

**Step three, Refactor phase — verify.** Instead of running the full test suite, we only run tests related to git changes with a single command: `pnpm test:changed`. It uses the git diff plus module dependency graph to determine which tests are affected, running only those for fast feedback. All green means we move to the next cycle.

The key insight: in AI-assisted development, **tests are no longer just a quality assurance tool — they become the acceptance criteria you hand to AI**. You define "what correct looks like," and AI figures out "how to get there."

## Two Key Technical Choices

Test cases fall into two categories, each requiring a different technical approach.

### data-testid: Stable Anchors for Structural Assertions

The first category is structural and interaction assertions. We use `data-testid` as the primary element locator. Why not text content? Because copy, icons, and translations change frequently during development. `data-testid` is a stable anchor that won't break tests just because someone changed a word in the UI.

```tsx
// Text-based assertion: breaks when locale changes
expect(screen.getByText('Development Tools')).toBeInTheDocument()

// data-testid assertion: stable regardless of language or copy changes
expect(screen.getByTestId('category-tab-dev')).toBeInTheDocument()
```

`data-testid` is just a regular HTML attribute with zero performance impact. You can strip it in production builds with a babel plugin, but we chose to keep it — the QA team reuses the same testids for E2E testing, eliminating the need to maintain two sets of locator strategies.

An additional benefit: pure UI refactors (like splitting one large component into two smaller ones) don't require test changes. As long as the `data-testid` attributes remain, tests don't care how the component hierarchy changes.

### MSW: Assertions on API Data

The second category is API data assertions. We use MSW to mock API responses with constructed data, then verify that the data renders correctly on the page. For example, if the API returns "Skill 1," the page should display that text; when navigating to page two, the request parameters should include `page=2`.

We write component-level unit tests — each test file corresponds to one component, with MSW mocking its API dependencies. The test boundary is the component itself. What we verify is "given certain inputs, what behavior should the component exhibit": props-driven rendering, state changes after user interactions, and API data-to-UI mapping.

## Case Study 1: Category API SSR Refactoring

The first case is an SSR migration of the category API — a refactoring scenario. Skills Hub's category list was originally hardcoded on the frontend and needed to be fetched from a backend API with SSR prefetching. Meanwhile, the InstallCard component was making an API call on every tab switch and needed to switch to SSR-prefetched data with local tab switching. A total of 6 files needed coordinated changes.

### Red Phase

I provided the requirements and API documentation to AI in TDD mode. AI only touched test files, leaving business code untouched. It added 4 new test cases:

```tsx
// views-index.test.tsx
it('SSR category data renders correctly as tab list', () => {
  render(<ViewsIndex categories={mockCategories} />)
  expect(screen.getByTestId('category-tab-dev')).toBeInTheDocument()
  expect(screen.getByTestId('category-tab-design')).toBeInTheDocument()
})

it('does not render tabs when categories are not provided', () => {
  render(<ViewsIndex categories={[]} />)
  expect(screen.queryByTestId('category-tabs')).not.toBeInTheDocument()
})
```

InstallCard's tests were completely rewritten — from MSW-based API interception to direct props-driven assertions, since after the refactor, InstallCard no longer makes its own requests but receives data prefetched by its parent component via SSR:

```tsx
// Before refactor: InstallCard internally uses useSWR
server.use(http.get('/api/extensions', () => { ... }))

// After refactor: InstallCard becomes a pure display component
render(<InstallCard extensions={mockExtensions} />)
expect(screen.getByTestId('install-card-vscode')).toBeInTheDocument()
```

Run the tests — widespread red, as expected.

### Green Phase

AI modified the 6 files in dependency order:

1. API layer: added type definitions and request functions
2. Constants file: removed the hardcoded category list
3. ViewsIndex: reads category data from props
4. InstallCard: removed `useSWR`, switched to props-driven
5. SSR page: `getServerSideProps` uses `Promise.all` to prefetch 3 APIs in parallel
6. Type file: updated page props types

A critical point here: InstallCard went from making internal requests to being purely props-driven. Without test protection, this kind of change is easy to get wrong. But since tests had already defined the expected behavior, AI just needed to make them pass — the direction couldn't drift.

### Verify Phase

`pnpm test:changed` — 88 tests all green, under one hour.

## Case Study 2: Sort Select — Adding a New Feature

The second case is adding a sort dropdown — a feature addition scenario. The requirement was to add a sort select next to the view toggle button, with options for default, favorites, installs, and name, passing the sort parameter to the search API.

### Red Phase

AI wrote 4 tests:

```tsx
it('sort select exists with default value', () => {
  render(<ViewsIndex {...defaultProps} />)
  const select = screen.getByTestId('sort-select')
  expect(select).toHaveValue('default')
})

it('sort select shares container with view toggle', () => {
  render(<ViewsIndex {...defaultProps} />)
  const toolbar = screen.getByTestId('view-toolbar')
  expect(within(toolbar).getByTestId('sort-select')).toBeInTheDocument()
  expect(within(toolbar).getByTestId('view-toggle')).toBeInTheDocument()
})

it('changing sort sends sort parameter', async () => {
  render(<ViewsIndex {...defaultProps} />)
  await userEvent.selectOptions(screen.getByTestId('sort-select'), 'installs')
  // Verify API is called with sort=installs parameter
})

it('changing sort resets page to 1', async () => {
  render(<ViewsIndex {...defaultProps} />)
  // Navigate to page 2, then change sort, verify page resets to 1
})
```

The interesting part of this case is that a problem was discovered while writing the tests. In our test environment, gate-ui's `Select` component is mocked using a native `<select>` element. But the native select's `onChange` passes a `ChangeEvent`, while the real gate-ui `Select`'s `onChange` passes the value string directly. If this discrepancy hadn't been caught in the Red phase, the implementation would have worked in tests but behaved incorrectly in the real environment. So we fixed the mock in the Red phase to match the real component's behavior.

### Green Phase

Only 2 business files were modified: constants file with sort option definitions, and the view component with new state, dropdown, and API parameter passing.

### Verify Phase

`pnpm test:changed` — 106 tests all green.

## Multi-Agent Parallel Development

An additional benefit of test coverage: you can run two or three AI Agents working on different modules simultaneously.

Changes must pass existing tests — breakages trigger immediate red, no manual review fallback needed. With robust test coverage, each Agent runs `pnpm test:changed` after completing its changes. Green means no cross-contamination.

Conflict detection relies on two layers:

- **Same-file conflicts**: handled automatically by git merge
- **Cross-file behavioral conflicts**: `test:changed` uses dependency graph analysis. If Agent A modifies a component's props structure, and Agent B's module imports that component, Agent B's tests are automatically included in the `test:changed` execution scope

## Why It's Worth Doing

Three benefits. First, Red-first forces you to think through requirements. If you let AI go straight from requirements to code, it can easily drift in a direction that's hard to spot. Writing tests first means you must clarify what the feature should actually do — tests become executable requirements documentation. Second, AI's changes become controllable. Code must pass existing tests after changes — breakages surface immediately. Third, refactoring becomes safe. When you need to extract hooks or restructure components, passing tests means you can merge with confidence.

On the quality side, after running this project with the workflow, functional bugs at the QA stage were very few. Feedback was mostly about UI details — spacing, colors — not logic errors. Additionally, the tests themselves serve as behavioral documentation — new team members can read tests to understand what a component should do.

## Costs and Limitations

To be honest about the costs: the ratio of test code to business code is roughly 1.15:1, which is additional investment. But AI generates tests quickly, so this cost is primarily borne by AI.

On limitations, this approach works well for component development and refactoring with clear requirements and API-driven interfaces. If requirements are still in flux and UI hasn't been finalized — the exploration phase — writing tests first becomes a burden. Pure styling work isn't a great fit either, since CSS changes are hard to cover effectively with assertions.

## Summary

Use tests to constrain AI, catching problems automatically at the coding stage rather than relying on someone to discover them later. Writing tests looks like an extra step, but it saves the time you'd otherwise spend tracking down bugs and reworking code.
