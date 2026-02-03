# GitHub Actions Step Output Issue: Understanding the `foo` vs `foo1` Problem

## Executive Summary

The issue you're experiencing is due to a **name mismatch** between the step output variable being **set** and the step output variable being **referenced**. This is a common mistake when renaming variables in GitHub Actions workflows.

## The Problem

When you changed `foo` to `foo1` in the second workflow file, you only changed it in **one location** but not in the **other locations** where it's being referenced.

---

## Detailed Analysis

### Working File: `03-core-features--06-passing-data.yaml`

Let's trace the flow of the `foo` variable:

#### Step 1: Setting the Step Output
```yaml
- name: Generate and export foo
  id: generate-foo
  run: |
    foo=bar
    # 1) Step output (for job output)
    echo "foo=${foo}" >> "$GITHUB_OUTPUT"  # ← Creates output named "foo"
```
**Result:** Creates a step output accessible as `steps.generate-foo.outputs.foo`

#### Step 2: Declaring Job Output
```yaml
outputs:
  foo: ${{ steps.generate-foo.outputs.foo }}  # ← References "foo"
```
**Result:** Job output `foo` gets the value from step output `foo`

#### Step 3: Using in Same Job
```yaml
- name: Inspect values inside producer
  run: |
    echo "foo (step output):          ${{ steps.generate-foo.outputs.foo }}"  # ← References "foo"
```
**Result:** Successfully prints "bar" because `steps.generate-foo.outputs.foo` exists

---

### Not Working File: `03-core-features--06-passing-data-2.yaml`

Now let's see where it breaks:

#### Step 1: Setting the Step Output (CHANGED)
```yaml
- name: Generate and export foo
  id: generate-foo
  run: |
    foo=bar
    # 1) Step output (for job output)
    echo "foo1=${foo}" >> "$GITHUB_OUTPUT"  # ← Creates output named "foo1" (CHANGED!)
```
**Result:** Creates a step output accessible as `steps.generate-foo.outputs.foo1` (NOT `.foo`)

#### Step 2: Declaring Job Output (NOT CHANGED)
```yaml
outputs:
  foo: ${{ steps.generate-foo.outputs.foo }}  # ← Still references "foo" (MISMATCH!)
```
**Result:** Job output `foo` tries to get value from `steps.generate-foo.outputs.foo`, but this doesn't exist! The actual output is named `foo1`.

#### Step 3: Using in Same Job (PARTIALLY CHANGED)
```yaml
- name: Inspect values inside producer
  run: |
    echo "foo1 (step output):          ${{ steps.generate-foo.outputs.foo }}"  # ← Display text changed to "foo1", but still references "foo" (MISMATCH!)
```
**Result:** Tries to access `steps.generate-foo.outputs.foo` which doesn't exist, prints empty string

---

## Why This Happens

GitHub Actions step outputs work like a key-value store:

```
GITHUB_OUTPUT File Contents:
┌──────────────────────────┐
│ foo1=bar                 │  ← This is what you're writing
└──────────────────────────┘

Accessible As:
┌──────────────────────────────────────────┐
│ steps.generate-foo.outputs.foo1 = "bar"  │  ← This is created
│ steps.generate-foo.outputs.foo  = ""     │  ← This doesn't exist (empty)
└──────────────────────────────────────────┘
```

When you write `echo "foo1=${foo}" >> "$GITHUB_OUTPUT"`, you're creating an output with the **name** `foo1`, not `foo`.

---

## The Fix (For Reference Only - Not Making Changes)

To make the second workflow work correctly, you would need to change **all three locations**:

### Location 1: Setting the Output
```yaml
echo "foo1=${foo}" >> "$GITHUB_OUTPUT"  # ✓ Already changed
```

### Location 2: Job Outputs Declaration (NEEDS CHANGE)
```yaml
outputs:
  foo: ${{ steps.generate-foo.outputs.foo1 }}  # ← Change .foo to .foo1
  # OR rename the job output itself:
  # foo1: ${{ steps.generate-foo.outputs.foo1 }}
```

### Location 3: Reference in Echo Statement (NEEDS CHANGE)
```yaml
echo "foo1 (step output):          ${{ steps.generate-foo.outputs.foo1 }}"  # ← Change .foo to .foo1
```

---

## Key Concepts to Understand

### 1. Step Outputs vs Environment Variables

These are **two different mechanisms** in GitHub Actions:

| Mechanism | Set Using | Access Within Same Job | Access From Other Jobs |
|-----------|-----------|------------------------|------------------------|
| **Step Output** | `echo "name=value" >> "$GITHUB_OUTPUT"` | `${{ steps.step-id.outputs.name }}` | Via job outputs: `${{ needs.job-name.outputs.name }}` |
| **Environment Variable** | `echo "NAME=value" >> "$GITHUB_ENV"` | `$NAME` or `${{ env.NAME }}` | ❌ Not accessible (job-scoped only) |

### 2. The Output Name Must Match

The syntax `echo "foo1=bar" >> "$GITHUB_OUTPUT"` creates an output where:
- **Output name** = `foo1` (the part before `=`)
- **Output value** = `bar` (the part after `=`)

To access this, you MUST use the exact name: `steps.step-id.outputs.foo1`

### 3. Why FOO Works But foo Doesn't

In your output, you see:
```
FOO (set via GITHUB_ENV):   bar     ← This works
foo1 (step output):                  ← This is empty
```

**Why FOO works:**
- `FOO` is set via `GITHUB_ENV` on line 18: `echo "FOO=${foo}" >> "$GITHUB_ENV"`
- This creates an environment variable that's accessible in all subsequent steps within the same job
- You didn't change this line, so it still works

**Why foo1 step output is empty:**
- You're trying to access `steps.generate-foo.outputs.foo` 
- But you actually created `steps.generate-foo.outputs.foo1`
- The names don't match, so GitHub Actions returns an empty string

---

## Visualization of the Issue

```
┌─────────────────────────────────────────────────────────────┐
│  What You Created:                                          │
│  ━━━━━━━━━━━━━━━━━                                          │
│  echo "foo1=bar" >> "$GITHUB_OUTPUT"                        │
│         ↓                                                   │
│  steps.generate-foo.outputs.foo1 = "bar"                    │
└─────────────────────────────────────────────────────────────┘
                              ↓
                              ↓ (mismatch!)
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  What You're Trying to Access:                              │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━                              │
│  ${{ steps.generate-foo.outputs.foo }}                      │
│         ↓                                                   │
│  steps.generate-foo.outputs.foo = "" (doesn't exist!)       │
└─────────────────────────────────────────────────────────────┐
```

---

## Conclusion

The issue is a simple but important lesson in GitHub Actions:

**The name you use when setting an output (`foo1`) MUST match the name you use when accessing it (`.outputs.foo1`).**

In your second workflow:
- ✓ You changed the **name** when **setting** the output to `foo1`
- ✗ You didn't change the **name** when **accessing** the output (still using `foo`)
- Result: GitHub Actions can't find an output named `foo`, returns empty string

This is why even a simple rename requires updating **all references** to that variable throughout the workflow file.

---

## Additional Notes

- GitHub Actions will **not** throw an error if you reference a non-existent output
- Instead, it silently returns an empty string
- This can make debugging challenging, so always double-check variable names when refactoring
- Use consistent naming to avoid confusion between step outputs, job outputs, and environment variables