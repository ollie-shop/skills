# Specs for Complexity Rules

How to use Specification-Driven Development (SDD) + tests to validate that components/functions follow complexity rules.

**Relationship:** This document implements rules from [Complexity Validation Rules](complexity-validation-rules.md). Start there to understand what limits apply, then use this guide to test them.

---

## Rule → Spec Mapping

### Components (React)

| Complexity Rule | Spec (Given/When/Then) | Automated Test |
|---|---|---|
| **useState ≤ 2** | Given a component, When mounted, Then should have ≤ 2 state vars | `expect(sourceCode).toMatch(/useState.*useState.*[^U]/) → FAIL` |
| **useEffect ≤ 1** | Given a component, When mounted, Then should have ≤ 1 effect | `sourceCode.match(/useEffect/g)?.length <= 1` |
| **Nesting ≤ 1 level** | Given logic in useEffect, When parsing, Then no `if { if { } }` | `countNestingLevels(parseEffect()) <= 1` |
| **Derived booleans ≤ 2** | Given variables, When declared, Then inline booleans ≤ 2 | `sourceCode.match(/const is\w+ =/g)?.length <= 2` |
| **Render < 100 lines** | Given a component, When rendered, Then JSX < 100 lines | `getJSXLineCount() < 100` |
| **Defensive parsing isolated** | Given external input, When used, Then extracted to function | `sourceCode.match(/as \w+\)?\?/) → must be in helpers/` |
| **Descriptive names** | Given variables, When read, Then names are self-documenting | `sourceCode.includes('const cb =') → FAIL; 'const cashback =' → OK` |

---

## Real-world Example: B2BPaymentTerms

### Specs in Given/When/Then Format

```gherkin
Feature: B2BPaymentTerms - Eligibility & Sync
  
  Scenario: PF customer should not see B2B terms
    Given a PF customer with salesChannel = "1"
    When the component mounts
    Then component is hidden (returns only children)
    And save() is never called
  
  Scenario: PJ customer (CNPJ) should see B2B terms
    Given a customer with valid CNPJ document
    When the component mounts
    Then component is visible
    And select shows payment term options
  
  Scenario: Auto-persist default only once
    Given a B2B customer with no persisted payment term
    And autoPersistDefault = true
    When component mounts
    Then save() is called exactly once with firstOption
    And subsequent mounts do NOT call save() again
```

### Tests Validating Complexity + Behavior

```typescript
describe("B2BPaymentTerms", () => {
  // 1. Complexity validation
  it("should extract state sync to custom hook (0 useState in main)", () => {
    expect(component).toHaveSourceCode(/useSyncPaymentTerm/);
    expect(component).not.toHaveSourceCode(/useState/);
  });

  // 2. Behavior validation (Given/When/Then)
  it("Given a PJ customer, When mount, Then renders select", () => {
    const { getByRole } = render(<B2BPaymentTerms ... />);
    expect(getByRole("combobox")).toBeInTheDocument();
  });

  it("Given autoPersistDefault=true, When session loads, Then saves once", () => {
    const mockSave = jest.fn();
    const { rerender } = render(<B2BPaymentTerms save={mockSave} />);
    expect(mockSave).toHaveBeenCalledTimes(1);
    
    rerender(<B2BPaymentTerms save={mockSave} />);
    expect(mockSave).toHaveBeenCalledTimes(1); // still 1, not 2
  });

  // 3. Code quality validation
  it("should use descriptive variable names (not abbreviations)", () => {
    expect(component).toHaveSourceCode(/const isVisible/);
    expect(component).toHaveSourceCode(/const days/);
    expect(component).not.toHaveSourceCode(/const cb =/);
  });
});
```

---

## Real-world Example: CashbackPaymentPanel

### When to Split into Sub-components (Decision Rule)

**Splitting makes sense if:**
- ✅ 3+ distinct visual states (applied ≠ below-minimum ≠ available)
- ✅ Each state has its own logic (handlers, hooks)
- ✅ Saves **≥ 30 lines** in the main component
- ✅ Each sub-component stays **< 40 lines** (testable)

**Do NOT split if:**
- ❌ Only 1-2 simple states
- ❌ Logic is minimal (one if, one handler)
- ❌ Sub-components would be **> 30 lines** (overhead > benefit)

**CashbackPaymentPanel:** ✅ Split (158 → 13 main + 3 subs of 20-28 lines = gain)  
**B2BPaymentTerms:** ❌ Did not split (stayed 63 lines, clear enough)

### Specs in Given/When/Then Format

```gherkin
Feature: CashbackPaymentPanel - State Visibility & Actions
  
  Scenario: Applied cashback shows delete button
    Given a customer with applied cashback covering 50% of order
    When component renders
    Then see checkmark + applied amount + delete button
    And remaining balance shown
  
  Scenario: Below minimum shows informational message
    Given a customer with balance < minimum purchase
    When component renders
    Then see banner with "Purchase from $X"
    And NO apply button (informational only)
  
  Scenario: Available cashback shows apply button
    Given a customer with usable cashback
    When user clicks "Apply"
    Then updateGiftCards called with [otherCards + cashback]
    And success message shown
```

### Tests Validating Complexity + Visibility

```typescript
describe("CashbackPaymentPanel", () => {
  // 1. Complexity validation
  it("index.tsx should have 0 useState (state in hooks)", () => {
    expect(indexTsx).not.toHaveSourceCode(/useState/);
    expect(indexTsx).toHaveSourceCode(/useCashback/);
  });

  it("should NOT pass > 0 props to sub-components (hook isolation)", () => {
    // Instead of: <CashbackApplied appliedCents={...} />
    // Expect: <CashbackApplied /> (calls useCashback itself)
    expect(render).toHaveJSX(/<CashbackApplied \/>/);
  });

  // 2. Behavior validation (Given/When/Then)
  it("Given applied cashback, When render, Then show checkmark", () => {
    const { getByRole } = render(<CashbackPaymentPanel />);
    expect(getByRole("img", { hidden: true })).toHaveAttribute("data-icon", "check");
  });

  it("Given below minimum, When render, Then NO apply button", () => {
    const { queryByText } = render(<CashbackPaymentPanel belowMinimumPurchase />);
    expect(queryByText("Apply")).not.toBeInTheDocument();
  });

  // 3. Sub-component isolation
  it("CashbackApplied should call useCashback (no props)", () => {
    expect(CashbackApplied).toHaveSourceCode(/useCashback\(\)/);
    expect(CashbackApplied).not.toHaveSourceCode(/interface.*Props/);
  });
});
```

---

## CI Validation Pipeline

```bash
# 1. Complexity checks (automated)
npm run lint:complexity
  ├─ Parse AST: count useState, useEffect, nesting, derived booleans
  ├─ Lint: var names (reject cb, data, x)
  └─ Report: "✅ 0 violations" or "❌ useState=3, max=2"

# 2. Type checks
npm run type-check
  └─ tsc --noEmit

# 3. Unit tests (behavior + complexity)
npm run test
  ├─ Given/When/Then behavior specs
  ├─ Complexity source-code assertions
  └─ Code quality (naming, isolation)

# 4. Integration tests (optional, for external API calls)
npm run test:integration
```

---

## Pre-production Checklist

1. **Write spec first** (Given/When/Then):
   ```gherkin
   Given [state], When [action], Then [result]
   ```

2. **Generate component** from spec (via AI or manual implementation)

3. **Validate against complexity rules** (from [Complexity Validation Rules](complexity-validation-rules.md)):
   - ✅ useState ≤ 2
   - ✅ useEffect ≤ 1
   - ✅ nesting ≤ 1
   - ✅ derived booleans ≤ 2
   - ✅ render < 100 lines
   - ✅ descriptive names

4. **Run tests**:
   ```bash
   npm test
   ```
   - PASS: accept (complexity + behavior validated)
   - FAIL: refactor and retry

5. **No manual code review** (tests are the gate)

---

## Why This Works

- **Specs** define expected behavior (what, not how)
- **Tests** validate both behavior and complexity
- **AI** generates clean code because constraints are clear
- **Metrics** are objective (not "looks good?" but "useState = 1, OK")
- **No tech debt** because complexity violations are caught early

---

## Next Steps

1. Add `spec.ts` files to EVERY new component (required before merge)
2. Build custom linter: `eslint-plugin-complexity-rules`
3. Run in CI: fail PR if rules are violated or tests fail
