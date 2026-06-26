# References Guide

This directory contains the canonical standards for building checkout components. The documents form a three-layer governance model:

---

## 📋 Three-Layer Model

### Layer 1: **What** → [Complexity Validation Rules](complexity-validation-rules.md)

**Purpose:** Define metric-based limits that prevent cognitive overload.

**What you get:**
- Quantitative rules (`useState ≤ 2`, `useEffect ≤ 1`, etc.)
- Rationale with before/after code examples
- Self-assessment checklist for PRs

**When to use:**
- During code review: "Does this violate a complexity limit?"
- Before submitting a PR: Use the checklist to self-assess
- When refactoring: Understand what "simple" means

**Key sections:**
- Components (React): state/effect/nesting/naming limits
- Hub Functions: error handling, mutation guards
- Actions: sensitive data, fetch safety, logging
- PR Checklist: self-assessment gates

---

### Layer 2: **How to Test** → [Specs for Complexity Rules](specs-for-complexity-rules.md)

**Purpose:** Automate complexity validation via Specification-Driven Development (SDD).

**What you get:**
- Gherkin Given/When/Then templates
- Automated test patterns (Jest + AST parsing)
- Real-world examples (B2BPaymentTerms, CashbackPaymentPanel)
- CI/CD pipeline integration

**When to use:**
- Before writing code: Write the spec (Given/When/Then)
- During implementation: Use test templates to validate both behavior and complexity
- In CI: Automated gates that fail PRs on rule violations

**Key sections:**
- Rule → Spec Mapping: Maps each complexity rule to a testable spec
- Real-world examples: Full Gherkin + Jest code for real components
- CI pipeline: `npm run lint:complexity` + `npm test`
- Pre-production checklist: Gates before merge

---

### Layer 3: **How to Build** → [Coding Standards](coding-standards.md)

**Purpose:** SDK-specific standards and safety nets for component implementation.

**What you get:**
- TypeScript patterns (no class components, default exports, etc.)
- Naming conventions (PascalCase, `on<Action>`, `is<State>`)
- CSS class naming and design tokens
- Loading state constraints (hydration safety, CSS scoping)
- REQUEST guard patterns (validation, error handling, unwrap progressively)
- Observability (logging prefix, error extraction)

**When to use:**
- During development: Follow these patterns while building
- Before handoff: Use the Definition of Done checklist
- When integrating with SDK: Understand REQUEST validation, error callbacks, unmount safety

**Key sections:**
- TypeScript & SDK: Hook/state/effect guidance with complexity limits
- Naming & prop conventions: Consistent naming across the codebase
- Loading states: Hydration mismatches, CSS keyframe scoping
- Guards on REQUEST: Validation, error handling, progressive unwrap
- Definition of done: Pre-handoff checklist

---

## 🔄 Workflow: Design → Build → Test → Merge

### Phase 1: **Understand Limits** (Read Layer 1)

1. Open [Complexity Validation Rules](complexity-validation-rules.md)
2. Find your component type (React, Hub, Action)
3. Note the metric limits and rationale
4. Bookmark the PR checklist

---

### Phase 2: **Write Spec** (Read Layer 2, Write Gherkin)

1. Open [Specs for Complexity Rules](specs-for-complexity-rules.md)
2. Use the Given/When/Then template for your component
3. Write scenarios that cover main paths + edge cases

```gherkin
Feature: ComponentName - Feature
  
  Scenario: User action triggers expected result
    Given [initial state]
    When [user action]
    Then [expected outcome]
```

---

### Phase 3: **Build Component** (Read Layer 3)

1. Open [Coding Standards](coding-standards.md)
2. Follow TypeScript, naming, and SDK patterns
3. Implement component, keeping [Complexity Validation Rules](complexity-validation-rules.md) limits in mind
4. Watch for cross-references that link to other docs

---

### Phase 4: **Write Tests** (Read Layer 2, Write Jest)

1. Open [Specs for Complexity Rules](specs-for-complexity-rules.md)
2. Use the test templates for your component type
3. Add both behavior tests (Given/When/Then) and complexity assertions (code inspection)

```typescript
it("should have ≤ 2 useState hooks", () => {
  expect(component).not.toHaveSourceCode(/useState.*useState.*useState/);
});
```

---

### Phase 5: **Validate Before Merge** (Layer 1 Checklist)

1. Open [Complexity Validation Rules](complexity-validation-rules.md) § PR Checklist
2. Go through each item
3. If all ✅, ready for merge
4. If ❌, refactor and re-test

---

## 🎯 Quick Reference: Find What You Need

| Question | Document | Section |
|----------|----------|---------|
| "What's the useState limit?" | Layer 1 | Components (React) |
| "How do I test the useState limit?" | Layer 2 | Rule → Spec Mapping |
| "Should I use `useState` or SDK state?" | Layer 3 | TypeScript & SDK |
| "How do I name variables?" | Layer 3 | Naming & prop conventions |
| "What's the comment rule?" | Layer 1 | Comments & Naming |
| "How do I handle REQUEST errors?" | Layer 3 | Guards on REQUEST |
| "Why are loading states tricky?" | Layer 3 | Loading states |
| "How do I log?" | Layer 1 | Actions § Log statements |
| "How do I validate my component?" | Layer 1 | PR Checklist |
| "How do I write specs?" | Layer 2 | Real-world Examples |

---

## 📌 Cross-Reference Map

**Complexity Validation Rules** links to:
- → Specs for Complexity Rules: "See this for automated testing patterns"
- → Coding Standards: "See this for SDK-specific guidance"

**Specs for Complexity Rules** links to:
- → Complexity Validation Rules: "Validates these metric limits"

**Coding Standards** links to:
- → Complexity Validation Rules: "For useState/useEffect limits, unwrap progressively, logging prefix"
- → Specs for Complexity Rules: "For test patterns and examples"

---

## ✅ Definition of Done

A component is ready for merge if:

1. **Code passes Layer 1 checklist** (Complexity Validation Rules)
   - ✅ useState ≤ 2
   - ✅ useEffect ≤ 1
   - ✅ Nesting ≤ 1
   - ✅ Derived booleans ≤ 2
   - ✅ Render < 100 lines
   - ✅ Names are self-documenting

2. **Tests pass Layer 2 gates** (Specs for Complexity Rules)
   - ✅ Behavior tests (Given/When/Then)
   - ✅ Complexity assertions (AST checks)
   - ✅ Code quality checks (no abbreviations, no raw casting)

3. **Implementation follows Layer 3 standards** (Coding Standards)
   - ✅ TypeScript + SDK patterns correct
   - ✅ Naming conventions followed
   - ✅ Loading/error states safe
   - ✅ REQUEST guards in place
   - ✅ Logging with prefix

4. **Definition of done from Coding Standards** is checked (line ~75)

---

## 🚀 Making Changes to This Guide

If you update one document, check if the others need updates:

- **Change to Layer 1 (rules)?** → Update Layer 2 (specs) and Layer 3 (standards)
- **Add new pattern to Layer 3?** → Consider if it should be in Layer 1 as a rule
- **Add new test pattern to Layer 2?** → Ensure it validates a Layer 1 rule

Keep the three documents in sync. They form a unified governance model.
