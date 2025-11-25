# ⚠️ SUPERSTITION: Don't Use Default<> in Nested Interface Properties

**⚠️ WARNING: This is a SUPERSTITION - unverified folk knowledge from a single observation.**

This may be wrong, incomplete, or context-specific. Use with extreme skepticism and verify thoroughly!

## Topic

Using `Default<T, V>` in nested interface properties when pushing to Cell arrays

## Problem

When you have an interface with `Default<>` wrappers on properties, and that interface is used as an array element type, pushing new elements to the array causes TypeScript errors about opaque cell types.

### What Didn't Work

```typescript
interface Dimension {
  name: string;
  multiplier: Default<number, 1>;           // ❌ Default in nested property
  categories: Default<CategoryOption[], []>; // ❌ Default in nested property
}

interface RubricInput {
  dimensions: Default<Dimension[], []>;      // ✅ Default at array level is fine
}

// In handler:
const addDimension = handler<unknown, { dimensions: Cell<Dimension[]> }>(
  (_, { dimensions }) => {
    dimensions.push({
      name: "New Dimension",
      multiplier: 1,           // ❌ TypeScript error here!
      categories: [],          // ❌ TypeScript error here!
    });
  }
);
```

**Symptom:** TypeScript errors like:
```
Type 'number' is not assignable to type 'OpaqueCell<number> & number'
Type 'string' is not assignable to type 'OpaqueCell<string> & string'
```

**Why:** When pushing to a `Cell<T[]>`, TypeScript expects the pushed values to match the Cell-wrapped types from the interface. `Default<>` at the property level creates nested Cell wrappers that conflict with plain values.

## Solution That Seemed to Work

**Only use `Default<>` at the array level**, not on nested properties. Provide defaults when pushing:

```typescript
interface Dimension {
  name: string;
  multiplier: number;           // ✅ Plain type, no Default
  categories: CategoryOption[]; // ✅ Plain type, no Default
}

interface RubricInput {
  dimensions: Default<Dimension[], []>;  // ✅ Default only here
}

// In handler:
const addDimension = handler<unknown, { dimensions: Cell<Dimension[]> }>(
  (_, { dimensions }) => {
    dimensions.push({
      name: "New Dimension",
      multiplier: 1,           // ✅ Works! Provide default inline
      categories: [],          // ✅ Works! Provide default inline
    });
  }
);
```

**Result:** TypeScript is happy, values push correctly, no type errors.

## Context

- **Pattern:** smart-rubric.tsx (Phase 1 validation)
- **Use case:** Dynamically adding dimensions and options to arrays
- **Framework:** CommonTools with TypeScript
- **Error location:** Handler functions that push new elements to Cell arrays

## Theory / Hypothesis

The `Default<T, V>` wrapper is meant for **top-level pattern inputs**, not for nested object properties:

1. **Top-level `Default<>`:**
   - Tells the framework "if this field is missing, use this default"
   - Works at pattern initialization time
   - Creates a `Cell<T>` that starts with the default value

2. **Nested `Default<>` (problematic):**
   - Creates nested Cell wrappers: `Cell<Array<Cell<Dimension>>>`
   - When you `.push()` an object, TypeScript expects Cell-wrapped properties
   - You'd need to push `{ multiplier: cell(1) }` instead of `{ multiplier: 1 }`
   - This is cumbersome and likely not the intended pattern

**Mental model:**
- Use `Default<>` for **pattern inputs** (top-level fields)
- Use plain types for **nested properties**
- Provide defaults **inline when pushing** new elements

## Examples

### ❌ Don't use Default in nested properties:

```typescript
interface CategoryOption {
  label: string;
  score: Default<number, 0>;  // ❌ Causes push errors
}

interface Dimension {
  name: string;
  categories: Default<CategoryOption[], []>;  // ❌ Causes push errors
}

// Later trying to push:
dimensions.push({
  name: "Location",
  categories: [{ label: "Downtown", score: 10 }]  // ❌ Type error on score!
});
```

### ✅ Do use Default only at array level:

```typescript
interface CategoryOption {
  label: string;
  score: number;  // ✅ Plain type
}

interface Dimension {
  name: string;
  categories: CategoryOption[];  // ✅ Plain type
}

interface Input {
  dimensions: Default<Dimension[], []>;  // ✅ Default here only
}

// Later pushing works fine:
dimensions.push({
  name: "Location",
  categories: [{ label: "Downtown", score: 10 }]  // ✅ Works!
});
```

### ✅ Accessing nested Cell properties in handlers:

When you need to modify nested properties after they're in the array, use `.key()`:

```typescript
const changeDimensionMultiplier = handler<
  unknown,
  { dimension: Cell<Dimension>, delta: number }
>(
  (_, { dimension, delta }) => {
    const current = dimension.key("multiplier").get();  // ✅ Use .key() to access
    dimension.key("multiplier").set(current + delta);   // ✅ Use .key() to modify
  }
);
```

## Key Pattern Summary

```typescript
// Pattern structure:
interface NestedType {
  field1: PlainType;        // ✅ No Default here
  field2: PlainType;        // ✅ No Default here
}

interface PatternInput {
  arrayField: Default<NestedType[], []>;  // ✅ Default only at array level
}

// Pushing:
arrayField.push({
  field1: defaultValue1,    // ✅ Provide defaults inline
  field2: defaultValue2,    // ✅ Provide defaults inline
});

// Accessing after push:
element.key("field1").get()    // ✅ Use .key() to access
element.key("field1").set(val) // ✅ Use .key() to modify
```

## Related Patterns

- **Cell Property Access** - Using `.key()` to access nested Cell properties
- **Array Cell Operations** - Working with `Cell<Array<T>>`

## Metadata

```yaml
topic: typescript, Default, Cell, arrays, type-errors, nested-interfaces
discovered: 2025-11-24
confirmed_count: 1
last_confirmed: 2025-11-24
sessions: [smart-rubric-phase-1]
related_functions: Default, Cell, push, key
related_patterns: Cell-arrays, nested-objects
status: superstition
stars: ⭐⭐
```

## Guestbook

- ⭐⭐ 2025-11-24 - Hit this while building smart-rubric dynamic dimensions. Had `multiplier: Default<number, 1>` in the `Dimension` interface. When pushing new dimensions, got "Type 'number' is not assignable to type 'OpaqueCell<number> & number'" errors. Spent time trying to figure out if I needed to use `cell()` wrapper. Finally removed ALL `Default<>` from nested properties, kept it only at array level, provided defaults inline when pushing - worked immediately! Type errors gone. (smart-rubric-phase-1)

---

**Remember: This is just one observation. Test thoroughly in your own context!**

**TIP:** If you see "OpaqueCell & T" type errors when pushing to arrays, check if nested properties have `Default<>` wrappers - remove them and provide defaults inline instead.
