# Code Reviews and Pull Requests

## Table of Contents

- [Introduction](#introduction)
- [Code Review Principles](#code-review-principles)
- [Review Process](#review-process)
- [Pull Request Templates](#pull-request-templates)
- [Review Checklists](#review-checklists)
- [Best Practices](#best-practices)
- [Automation and Tools](#automation-and-tools)
- [Common Pitfalls](#common-pitfalls)
- [Team Culture](#team-culture)
- [Key Takeaways](#key-takeaways)

## Introduction

Code reviews are a critical practice in software development that ensure code quality, share knowledge, and maintain consistent standards across a team. When done well, they catch bugs early, improve code design, and help developers grow.

### Why Code Reviews Matter

```
Without Code Reviews:
Developer → Commit → Merge → Production → 🐛 Bug Found → Hotfix → Rollback
                                    ↑
                          Expensive fixes, customer impact

With Code Reviews:
Developer → PR → Review → Feedback → Fix → Merge → Production
                    ↑
              Catch issues early, learn and improve
```

### Benefits

- **Bug Detection**: Catch bugs before they reach production
- **Knowledge Sharing**: Spread understanding of codebase
- **Consistency**: Maintain coding standards across team
- **Mentorship**: Junior developers learn from seniors
- **Collective Ownership**: Team shares responsibility for code
- **Better Design**: Multiple perspectives improve solutions

## Code Review Principles

### The Four Core Principles

```
┌─────────────────────────────────────────┐
│   1. CONSTRUCTIVE FEEDBACK              │
│   Focus on code, not the person         │
│   "This function could be simplified"   │
│   NOT "You wrote bad code"              │
└─────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────┐
│   2. TIMELY REVIEWS                     │
│   Review within 24 hours                │
│   Unblock team members quickly          │
└─────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────┐
│   3. THOROUGH BUT PRAGMATIC             │
│   Balance quality with velocity         │
│   Don't block for nitpicks              │
└─────────────────────────────────────────┘
         ▼
┌─────────────────────────────────────────┐
│   4. LEARNING OPPORTUNITY               │
│   Explain your suggestions              │
│   Welcome questions and discussions     │
└─────────────────────────────────────────┘
```

### Review Mindset

```typescript
// ❌ Unhelpful Review
// "This is wrong"
// "Bad naming"
// "Why did you do this?"

// ✅ Constructive Review
/**
 * Consider extracting this logic into a separate function
 * for better testability and reusability.
 *
 * Example:
 * function validateUserInput(input: UserInput): ValidationResult {
 *   // validation logic here
 * }
 *
 * This would make the function easier to unit test and
 * could be reused in other components.
 */
```

### Feedback Categories

```typescript
// 🔴 Critical: Must be fixed before merge
// Security vulnerabilities, data loss risks, breaking changes

// 🟡 Major: Should be fixed before merge
// Performance issues, architectural concerns, complex bugs

// 🟢 Minor: Nice to have
// Style suggestions, alternative approaches, optimizations

// 💡 Nitpick: Optional, educational
// Naming improvements, formatting (if not automated)

// ❓ Question: Seeking clarification
// Understanding intent, edge cases, design decisions
```

## Review Process

### Standard PR Workflow

```
┌─────────────────────────────────────────┐
│  1. Developer creates feature branch    │
│     git checkout -b feature/new-feature │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  2. Implement feature with commits      │
│     - Write code                        │
│     - Add tests                         │
│     - Update docs                       │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  3. Self-review and cleanup             │
│     - Run linters                       │
│     - Run tests                         │
│     - Review own diff                   │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  4. Create pull request                 │
│     - Fill out PR template              │
│     - Add screenshots/videos            │
│     - Assign reviewers                  │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  5. Automated checks run                │
│     - CI/CD pipeline                    │
│     - Linting                           │
│     - Tests                             │
│     - Code coverage                     │
└─────────────────┬───────────────────────┘
                  │
                  ├─ Pass ─────────────────┐
                  │                         ▼
                  │               ┌──────────────────────┐
                  │               │  6. Code review      │
                  │               │     - Review code    │
                  │               │     - Leave comments │
                  │               │     - Approve/Request│
                  │               └─────────┬────────────┘
                  │                         │
                  │                         ├─ Approved ──┐
                  │                         │             ▼
                  │                         │    ┌──────────────┐
                  │                         │    │  8. Merge PR │
                  │                         │    └──────────────┘
                  │                         │
                  │                         └─ Changes Requested
                  │                                      │
                  └─ Fail ─────────────────────────────┐│
                                                        ││
                                                        ▼▼
                                          ┌──────────────────────┐
                                          │  7. Address feedback │
                                          │     - Fix issues     │
                                          │     - Push changes   │
                                          │     - Re-request     │
                                          └──────────┬───────────┘
                                                     │
                                                     └─► Back to step 5
```

### Self-Review Checklist

```markdown
Before creating a PR, review your own code:

## Functionality

- [ ] Feature works as intended
- [ ] Edge cases handled
- [ ] Error handling in place
- [ ] No console.logs or debug code

## Code Quality

- [ ] Code is readable and maintainable
- [ ] Functions are focused and single-purpose
- [ ] No code duplication
- [ ] Follows project conventions

## Testing

- [ ] Unit tests added/updated
- [ ] Tests cover edge cases
- [ ] All tests passing
- [ ] Test coverage maintained/improved

## Documentation

- [ ] Code comments for complex logic
- [ ] README updated if needed
- [ ] API docs updated
- [ ] CHANGELOG updated

## Performance

- [ ] No unnecessary re-renders (React)
- [ ] Efficient algorithms used
- [ ] No memory leaks
- [ ] Database queries optimized

## Security

- [ ] Input validation
- [ ] XSS prevention
- [ ] Authentication/authorization checks
- [ ] No sensitive data exposed
```

## Pull Request Templates

### Comprehensive PR Template

```markdown
## Description

<!-- Provide a clear and concise description of what this PR does -->

## Type of Change

<!-- Mark the relevant option with an 'x' -->

- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update
- [ ] Code refactoring
- [ ] Performance improvement
- [ ] Test addition or update

## Related Issues

<!-- Link to related issues -->

Closes #<!-- issue number -->
Fixes #<!-- issue number -->
Relates to #<!-- issue number -->

## Motivation and Context

<!-- Why is this change required? What problem does it solve? -->

## How Has This Been Tested?

<!-- Describe the tests that you ran to verify your changes -->

- [ ] Unit tests
- [ ] Integration tests
- [ ] E2E tests
- [ ] Manual testing

**Test Configuration**:

- Browser/OS:
- Node version:
- Database version:

## Screenshots (if applicable)

<!-- Add screenshots to help explain your changes -->

### Before

<!-- Screenshot or GIF of the old behavior -->

### After

<!-- Screenshot or GIF of the new behavior -->

## Checklist

<!-- Go through each item and mark completed items with an 'x' -->

### Code Quality

- [ ] My code follows the style guidelines of this project
- [ ] I have performed a self-review of my own code
- [ ] I have commented my code, particularly in hard-to-understand areas
- [ ] I have made corresponding changes to the documentation

### Testing

- [ ] I have added tests that prove my fix is effective or that my feature works
- [ ] New and existing unit tests pass locally with my changes
- [ ] Any dependent changes have been merged and published

### Documentation

- [ ] I have updated the README.md (if needed)
- [ ] I have updated the CHANGELOG.md
- [ ] I have added/updated JSDoc comments

### Dependencies

- [ ] I have checked that no new dependencies were added unnecessarily
- [ ] All new dependencies are necessary and documented

### Security

- [ ] I have checked for security vulnerabilities
- [ ] No sensitive data (keys, passwords, tokens) is exposed

### Performance

- [ ] I have considered the performance impact of my changes
- [ ] No performance regressions introduced

## Breaking Changes

<!-- List any breaking changes and migration steps -->

## Additional Notes

<!-- Any additional information for reviewers -->

## Reviewer Notes

<!-- Specific areas you'd like reviewers to focus on -->

---

/cc @reviewer1 @reviewer2
```

### Minimal PR Template

```markdown
## What does this PR do?

<!-- Brief description of changes -->

## Why?

<!-- Context and motivation -->

## How to test?

1.
2.
3.

## Screenshots

<!-- If applicable -->

## Checklist

- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] Self-reviewed

Closes #
```

### Bug Fix Template

```markdown
## Bug Description

<!-- What was the bug? -->

## Root Cause

<!-- What caused the bug? -->

## Solution

<!-- How does this fix it? -->

## Reproduction Steps (Before Fix)

1.
2.
3. **Expected**:
   **Actual**:

## Verification (After Fix)

<!-- How to verify the fix -->

## Related Issues

Fixes #

## Regression Testing

<!-- What areas might be affected? -->

- [ ] Tested area A
- [ ] Tested area B
- [ ] No regressions found
```

## Review Checklists

### General Code Review Checklist

```markdown
# Code Review Checklist

## Functionality

- [ ] Code does what it's supposed to do
- [ ] Edge cases are handled
- [ ] Error conditions are handled gracefully
- [ ] No breaking changes (or properly documented)

## Code Quality

- [ ] Code is readable and self-documenting
- [ ] Variable and function names are descriptive
- [ ] Functions are focused (single responsibility)
- [ ] No code duplication (DRY principle)
- [ ] Complexity is justified
- [ ] No commented-out code
- [ ] No unnecessary dependencies

## Architecture & Design

- [ ] Follows project architecture patterns
- [ ] Proper separation of concerns
- [ ] Reusable components where appropriate
- [ ] Appropriate abstraction level
- [ ] No premature optimization

## Testing

- [ ] Sufficient test coverage (>80% for new code)
- [ ] Tests are meaningful and test behavior
- [ ] Edge cases are tested
- [ ] Mocks are used appropriately
- [ ] Tests are maintainable

## Performance

- [ ] No obvious performance issues
- [ ] Efficient algorithms chosen
- [ ] No unnecessary loops or operations
- [ ] Database queries optimized
- [ ] Large lists virtualized (if UI)
- [ ] Images optimized

## Security

- [ ] Input is validated and sanitized
- [ ] No SQL injection vulnerabilities
- [ ] No XSS vulnerabilities
- [ ] Authentication/authorization checked
- [ ] Sensitive data not logged
- [ ] Dependencies have no known vulnerabilities

## Documentation

- [ ] Complex logic is commented
- [ ] Public APIs documented
- [ ] README updated if needed
- [ ] Breaking changes documented
- [ ] Migration guide provided (if breaking changes)

## Compatibility

- [ ] Backwards compatible (or breaking change justified)
- [ ] Works across supported browsers
- [ ] Mobile responsive (if UI)
- [ ] Accessible (WCAG 2.1 AA)

## Git Hygiene

- [ ] Commit messages are clear
- [ ] Commits are logical and focused
- [ ] No merge commits in feature branch
- [ ] No files changed accidentally
- [ ] Proper branch naming
```

### React-Specific Checklist

```markdown
# React Code Review Checklist

## Components

- [ ] Component is functional (not class-based unless necessary)
- [ ] Component is focused (single responsibility)
- [ ] Props are properly typed with TypeScript
- [ ] Props are validated or have defaults
- [ ] No prop drilling (use Context if needed)

## Hooks

- [ ] Hooks used correctly (only at top level)
- [ ] Dependencies arrays are correct
- [ ] No unnecessary re-renders
- [ ] Custom hooks for reusable logic
- [ ] `useCallback` and `useMemo` used appropriately

## State Management

- [ ] State is co-located where it's used
- [ ] No derived state (compute from existing state)
- [ ] State updates are immutable
- [ ] Global state is justified

## Performance

- [ ] `React.memo` used for expensive components
- [ ] Large lists are virtualized
- [ ] Images are lazy-loaded
- [ ] Code splitting for large components
- [ ] No inline function definitions in JSX (if performance-critical)

## Accessibility

- [ ] Semantic HTML used
- [ ] ARIA attributes where needed
- [ ] Keyboard navigation works
- [ ] Focus management correct
- [ ] Screen reader tested

## Testing

- [ ] Components tested with React Testing Library
- [ ] User interactions tested
- [ ] Error states tested
- [ ] Loading states tested
```

### TypeScript-Specific Checklist

```markdown
# TypeScript Code Review Checklist

## Types

- [ ] No `any` types (use `unknown` if necessary)
- [ ] Proper type annotations for functions
- [ ] Return types specified explicitly
- [ ] Generics used appropriately
- [ ] Union types over enums (when applicable)

## Interfaces vs Types

- [ ] Interfaces for objects/classes
- [ ] Types for unions/intersections
- [ ] Consistent naming conventions

## Type Safety

- [ ] No type assertions (except when necessary)
- [ ] No `as any` casting
- [ ] Proper null/undefined handling
- [ ] Type guards used correctly

## Advanced Types

- [ ] Proper use of utility types (Pick, Omit, etc.)
- [ ] Conditional types used correctly
- [ ] Mapped types for transformations
```

## Best Practices

### 1. Review Small PRs

```
Small PR (< 400 lines):
- Takes 15-30 minutes to review
- Easy to understand
- Quick feedback cycle

Large PR (> 1000 lines):
- Takes hours to review
- Easy to miss bugs
- Reviewer fatigue
- Long feedback cycle
```

```markdown
# Guidelines for PR Size

## Ideal

- 200-400 lines changed
- Single feature or bug fix
- Reviewed in one sitting

## Split Large Changes

1. Create base PR with infrastructure
2. Create PRs for individual features
3. Create PR for integration

Example:

- PR #1: Add user model and database schema
- PR #2: Add user authentication endpoints
- PR #3: Add user profile UI
- PR #4: Integrate authentication with UI
```

### 2. Use Review Comments Effectively

````typescript
// ❌ Vague comment
// "This won't work"

// ✅ Specific and constructive
/**
 * 🔴 Critical: This will cause a memory leak
 *
 * The event listener is added in useEffect but never cleaned up.
 *
 * Suggested fix:
 * ```typescript
 * useEffect(() => {
 *   const handler = () => handleResize();
 *   window.addEventListener('resize', handler);
 *   return () => window.removeEventListener('resize', handler);
 * }, []);
 * ```
 */

// ❌ No context
// "Use useMemo here"

// ✅ Explain why
/**
 * 🟡 Performance: Consider using useMemo
 *
 * This calculation runs on every render. Since it depends only on
 * `data` prop, we can memoize it:
 *
 * ```typescript
 * const processedData = useMemo(
 *   () => data.map(item => transformItem(item)),
 *   [data]
 * );
 * ```
 *
 * This prevents unnecessary recalculations when unrelated state changes.
 */

// ❌ Personal preference without reason
// "I prefer forEach over map here"

// ✅ Explain or mark as nitpick
/**
 * 💡 Nitpick: `map` is more idiomatic for transformations
 *
 * Since we're creating a new array from the input, `map` makes
 * the intent clearer. But this is a minor preference.
 */
````

### 3. Approve vs Request Changes

```markdown
## Approve ✅

Use when:

- No blocking issues found
- Minor suggestions are optional
- Code meets quality standards

## Request Changes 🔴

Use when:

- Critical bugs present
- Security vulnerabilities
- Major architectural issues
- Tests failing

## Comment 💬

Use when:

- Asking questions
- Making suggestions
- Discussing alternatives
- No blocking issues
```

### 4. Respond to Feedback Constructively

```markdown
## As Author

✅ Good Responses:

- "Good catch! I'll fix that."
- "That's a good point. I'll extract this into a helper function."
- "I disagree because X, but I'm open to discussing further."
- "I didn't consider that edge case. Adding a test now."

❌ Bad Responses:

- "This works fine."
- "That's not important."
- "I don't have time for that."
- Ignoring feedback without response
```

### 5. Balance Speed and Quality

```markdown
## Fast-Track Criteria

Merge quickly when:

- [ ] Hotfix for production issue
- [ ] Blocking other team members
- [ ] Trivial change (typo fix, etc.)
- [ ] Time-sensitive feature

Fast-track process:

1. Add [URGENT] to PR title
2. Explain urgency in description
3. Request review in team chat
4. Get one approval (instead of two)

## Normal Review

Standard process for:

- [ ] New features
- [ ] Refactoring
- [ ] API changes
- [ ] Performance improvements
```

## Automation and Tools

### GitHub Actions for PR Checks

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: "18"
          cache: "npm"
      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: "18"
          cache: "npm"
      - run: npm ci
      - run: npm test -- --coverage
      - uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: "18"
          cache: "npm"
      - run: npm ci
      - run: npm run build

  size:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: andresz1/size-limit-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

### Automated PR Labeling

```yaml
# .github/workflows/label-pr.yml
name: Label PR

on:
  pull_request:
    types: [opened, edited, synchronize]

jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/labeler@v4
        with:
          configuration-path: .github/labeler.yml
```

```yaml
# .github/labeler.yml
"type: bug":
  - head-branch: ["^fix/", "fix"]

"type: feature":
  - head-branch: ["^feat/", "^feature/"]

"type: docs":
  - changed-files:
      - any-glob-to-any-file: ["**/*.md", "docs/**/*"]

"type: refactor":
  - head-branch: ["^refactor/"]

"area: frontend":
  - changed-files:
      - any-glob-to-any-file: ["src/components/**/*", "src/pages/**/*"]

"area: backend":
  - changed-files:
      - any-glob-to-any-file: ["api/**/*", "server/**/*"]

"size: small":
  - changed-files:
      - count-changed-files: "<10"

"size: large":
  - changed-files:
      - count-changed-files: ">100"
```

### Danger.js for Automated Reviews

```javascript
// dangerfile.js
import { danger, warn, fail, message } from "danger";

// PR Description
if (!danger.github.pr.body || danger.github.pr.body.length < 10) {
  fail("Please add a description to your PR.");
}

// PR Size
const bigPRThreshold = 600;
if (danger.github.pr.additions + danger.github.pr.deletions > bigPRThreshold) {
  warn(":exclamation: Big PR. Consider breaking it down into smaller PRs.");
}

// Tests Required
const hasAppChanges = danger.git.modified_files.some(
  (file) => file.startsWith("src/") && !file.includes(".test."),
);
const hasTestChanges = danger.git.modified_files.some((file) =>
  file.includes(".test."),
);

if (hasAppChanges && !hasTestChanges) {
  warn("This PR appears to be missing tests. Please add tests.");
}

// CHANGELOG
const hasChangelog = danger.git.modified_files.includes("CHANGELOG.md");
if (!hasChangelog) {
  warn("Please update CHANGELOG.md");
}

// Package.json changes
const packageChanged = danger.git.modified_files.includes("package.json");
const lockfileChanged = danger.git.modified_files.includes("package-lock.json");

if (packageChanged && !lockfileChanged) {
  warn(
    "package.json changed but package-lock.json did not. Please run npm install.",
  );
}

// Console.logs
const jsFiles = danger.git.created_files.filter(
  (file) =>
    file.endsWith(".js") || file.endsWith(".ts") || file.endsWith(".tsx"),
);

jsFiles.forEach(async (file) => {
  const content = await danger.github.utils.fileContents(file);
  if (content.includes("console.log")) {
    warn(`\`console.log\` found in ${file}. Please remove before merging.`);
  }
});

// No direct commits to main
if (
  danger.github.pr.base.ref === "main" &&
  danger.github.pr.head.ref === "main"
) {
  fail("Please use a feature branch. Do not commit directly to main.");
}

// Encourage smaller methods
const linesCount = danger.git.lines_of_code;
if (linesCount > 1000) {
  message(`This PR has ${linesCount} lines of code. Consider breaking it up.`);
}
```

## Common Pitfalls

### 1. Bike-shedding

```markdown
# Bike-shedding: Spending excessive time on trivial details

❌ Avoid:
"I prefer `forEach` over `map` here"
"Can we name this variable `userData` instead of `data`?"
"Should we use single quotes or double quotes?"

✅ Better:

- Automate style decisions with Prettier
- Focus on logic and architecture
- Save nitpicks for async discussions
```

### 2. Rubber-stamping

```markdown
# Rubber-stamping: Approving without proper review

❌ Avoid:

- "LGTM" without reading code
- Approving immediately
- Not testing functionality

✅ Better:

- Set aside dedicated time
- Review code thoroughly
- Ask questions
- Test critical changes locally
```

### 3. Too Many Reviewers

```markdown
# Too many reviewers slow down PRs

❌ Avoid:

- Requiring 5+ approvals
- Waiting for everyone to review

✅ Better:

- 1-2 reviewers for most PRs
- Subject matter expert for complex changes
- Team lead for architectural changes
```

## Team Culture

### Building a Positive Review Culture

```markdown
# Code Review Culture Guidelines

## Be Kind and Respectful

- Critique code, not people
- Assume good intent
- Say "we" not "you"

## Be Humble

- You might be wrong
- Learn from others
- Admit when you don't understand

## Be Explicit

- Categorize feedback (critical/major/minor/nitpick)
- Explain your reasoning
- Provide examples

## Be Responsive

- Review within 24 hours
- Respond to comments promptly
- Unblock teammates quickly

## Celebrate Good Code

- "👍 Nice solution!"
- "✨ This is a clever approach"
- "🎉 Great refactoring!"
```

### Handling Disagreements

```markdown
## When Author and Reviewer Disagree

### Step 1: Discussion

- Author explains their reasoning
- Reviewer explains their concerns
- Both seek to understand

### Step 2: Evidence

- Provide examples or data
- Show similar patterns in codebase
- Reference style guides or documentation

### Step 3: Compromise

- Find middle ground
- "Let's do X now, refactor to Y later"
- "Let's try it and monitor"

### Step 4: Escalation (if needed)

- Involve tech lead
- Discuss in team meeting
- Document decision
```

## Key Takeaways

1. **Constructive Feedback**: Focus on code, not people. Provide specific, actionable feedback with explanations and examples. Categorize by severity.

2. **Timely Reviews**: Review within 24 hours to unblock teammates. Set aside dedicated time for reviews rather than doing them when "you have a minute."

3. **Small PRs**: Keep PRs under 400 lines for effective review. Large PRs lead to reviewer fatigue and missed bugs. Split big changes into logical smaller PRs.

4. **Self-Review First**: Review your own code before requesting review. Run linters, tests, and check for debug code. Many issues can be caught before others review.

5. **Clear PR Descriptions**: Use templates to ensure consistency. Explain what, why, and how. Include screenshots for UI changes and testing instructions.

6. **Checklists**: Use review checklists for consistency. Cover functionality, code quality, testing, performance, security, and documentation.

7. **Automation**: Automate what you can (linting, testing, formatting). Let humans focus on logic, architecture, and design decisions.

8. **Balance Speed and Quality**: Not every PR needs the same scrutiny. Fast-track urgent fixes while maintaining standards for major features.

9. **Positive Culture**: Celebrate good code, be humble, and be kind. Code reviews are learning opportunities for everyone involved.

10. **Continuous Improvement**: Regularly retrospect on review process. What's working? What's slowing us down? Adapt processes based on team feedback.
