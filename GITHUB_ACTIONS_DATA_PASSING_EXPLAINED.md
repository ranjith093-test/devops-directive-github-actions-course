# GitHub Actions Data Passing: Complete Guide

## Table of Contents
1. [The Three Levels of Data Passing](#the-three-levels-of-data-passing)
2. [Understanding Your Confusion](#understanding-your-confusion)
3. [Step-by-Step Flow Visualization](#step-by-step-flow-visualization)
4. [Why Consumer Job Works](#why-consumer-job-works)
5. [The Independence Principle](#the-independence-principle)
6. [Complete Examples](#complete-examples)

---

## The Three Levels of Data Passing

GitHub Actions has **three distinct levels** of data passing, each with its own namespace and scope:

### Level 1: Shell Variables (Inside a Single Step)
```yaml
run: |
  foo=bar          # ← Shell variable (only exists in this step)
  echo $foo        # ← Prints "bar"
```
**Scope:** Only within the same `run` block
**Access:** `$variable_name` (bash syntax)

### Level 2: Step Outputs (Between Steps in Same Job)
```yaml
run: |
  echo "foo1=bar" >> "$GITHUB_OUTPUT"  # ← Creates step output named "foo1"
```
**Scope:** Accessible by other steps in the same job
**Access:** `${{ steps.step-id.outputs.foo1 }}`

### Level 3: Job Outputs (Between Different Jobs)
```yaml
outputs:
  foo: ${{ steps.generate-foo.outputs.foo1 }}  # ← Creates job output named "foo"
```
**Scope:** Accessible by other jobs that depend on this job
**Access:** `${{ needs.job-name.outputs.foo }}`

---

## Understanding Your Confusion

You're confused because you see **two different names** (`foo` and `foo1`) but the consumer job still works. Here's why:

### In Your Updated File 2:

```yaml
jobs:
  producer:
    outputs:
      foo: ${{ steps.generate-foo.outputs.foo1 }}  # ← JOB OUTPUT named "foo"
    steps:
    - name: Generate and export foo
      id: generate-foo
      run: |
        echo "foo1=${foo}" >> "$GITHUB_OUTPUT"     # ← STEP OUTPUT named "foo1"
```

**Key Insight:** These are **two separate things** with **two separate names**!

1. **Step Output** = `foo1` (Level 2)
2. **Job Output** = `foo` (Level 3)

---

## Step-by-Step Flow Visualization

Let's trace the data flow in your file 2:

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Shell Variable                                              │
│ ─────────────────────────                                           │
│ run: foo=bar                                                        │
│                                                                     │
│ Creates: Shell variable "foo" = "bar"                               │
│ Scope: This run block only                                          │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Create Step Output                                         │
│ ───────────────────────────                                         │
│ echo "foo1=${foo}" >> "$GITHUB_OUTPUT"                              │
│                                                                     │
│ Creates: Step output named "foo1" with value "bar"                  │
│ Accessible as: steps.generate-foo.outputs.foo1                      │
│ Scope: Other steps in the producer job                              │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: Create Job Output                                          │
│ ──────────────────────────                                          │
│ outputs:                                                            │
│   foo: ${{ steps.generate-foo.outputs.foo1 }}                       │
│                                                                     │
│ Creates: Job output named "foo" with value from step output "foo1"  │
│ Accessible as: needs.producer.outputs.foo                           │
│ Scope: Other jobs (like consumer)                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Why Consumer Job Works

### The Question:
> "In the consumer I am still referencing the old foo i.e `needs.producer.outputs.foo` but in the actions it is giving me correct output. This is really confusing"

### The Answer:

The consumer job references `needs.producer.outputs.foo`, which is **correct** because:

1. The **job output** is named `foo` (line 11 in your file 2)
2. Even though the **step output** is named `foo1`, the **job output** can have a different name
3. The job output `foo` gets its value from step output `foo1`

**Think of it like a relay race:**
```
Step Output "foo1" → Passes baton to → Job Output "foo" → Consumer receives as "foo"
     (Level 2)                              (Level 3)
```

### Visual Example from Your File 2:

```yaml
# INSIDE PRODUCER JOB:

# This creates STEP output named "foo1"
echo "foo1=bar" >> "$GITHUB_OUTPUT"

# This creates JOB output named "foo" (different name!)
outputs:
  foo: ${{ steps.generate-foo.outputs.foo1 }}
      ↑                                  ↑
   Job output                      Step output
   name is "foo"                   name is "foo1"

# INSIDE CONSUMER JOB:

# This accesses JOB output named "foo" (correct!)
echo "Value from producer: ${{ needs.producer.outputs.foo }}"
                                                        ↑
                                                Job output name
```

---

## The Independence Principle

**Key Concept:** Step output names and job output names are **completely independent** of each other!

You can have:
- Step output named `foo1` → Job output named `foo` ✓
- Step output named `foo` → Job output named `bar` ✓
- Step output named `my_step_value` → Job output named `result` ✓

### Analogy: Variable Assignment

Think of it like variable assignment in programming:

```javascript
// Step output (source)
let foo1 = "bar";

// Job output (destination - different name!)
let foo = foo1;

// Consumer uses the destination name
console.log(foo);  // Works! Prints "bar"
```

The consumer doesn't need to know about `foo1`. It only needs to know about `foo` (the job output).

---

## Complete Examples

### Example 1: Your Working File 1

```yaml
producer:
  outputs:
    foo: ${{ steps.generate-foo.outputs.foo }}  # Job output "foo" ← Step output "foo"
  steps:
  - id: generate-foo
    run: echo "foo=bar" >> "$GITHUB_OUTPUT"      # Step output "foo"

  - name: Use in same job
    run: echo "${{ steps.generate-foo.outputs.foo }}"  # Access step output "foo"

consumer:
  needs: producer
  steps:
  - run: echo "${{ needs.producer.outputs.foo }}"      # Access job output "foo"
```

**Flow:**
```
Shell var "foo" → Step output "foo" → Job output "foo" → Consumer gets "foo"
```

### Example 2: Your Updated File 2

```yaml
producer:
  outputs:
    foo: ${{ steps.generate-foo.outputs.foo1 }}  # Job output "foo" ← Step output "foo1"
  steps:
  - id: generate-foo
    run: echo "foo1=bar" >> "$GITHUB_OUTPUT"      # Step output "foo1"

  - name: Use in same job
    run: echo "${{ steps.generate-foo.outputs.foo1 }}"  # Access step output "foo1"

consumer:
  needs: producer
  steps:
  - run: echo "${{ needs.producer.outputs.foo }}"       # Access job output "foo"
```

**Flow:**
```
Shell var "foo" → Step output "foo1" → Job output "foo" → Consumer gets "foo"
                       ↑                      ↑                        ↑
                   Changed name          Different name           Uses job output name
```

### Example 3: Multiple Step Outputs → One Job Output

You can even have multiple step outputs combined into one job output:

```yaml
producer:
  outputs:
    result: ${{ steps.step1.outputs.part1 }}
  steps:
  - id: step1
    run: echo "part1=hello" >> "$GITHUB_OUTPUT"
  - id: step2
    run: echo "part2=world" >> "$GITHUB_OUTPUT"

consumer:
  needs: producer
  steps:
  - run: echo "${{ needs.producer.outputs.result }}"  # Gets "hello"
```

---

## Why This Design Makes Sense

### 1. Encapsulation
The consumer job doesn't need to know the internal details of the producer job (like step IDs or step output names). It only needs to know the **public interface** (job outputs).

### 2. Flexibility
You can refactor the producer job's internal implementation without breaking the consumer job:

```yaml
# Before: Step output named "foo"
outputs:
  foo: ${{ steps.generate-foo.outputs.foo }}

# After: Step output named "foo1" (internal change)
outputs:
  foo: ${{ steps.generate-foo.outputs.foo1 }}  # Consumer still uses "foo"
```

### 3. Clean API
Job outputs define a clear contract between jobs:

```yaml
producer:
  outputs:
    result: ...     # Public API
    status: ...     # Public API
    message: ...    # Public API
```

Consumers only need to know these output names, not the internal step structure.

---

## Common Pitfalls

### Pitfall 1: Confusing Step Output Name with Job Output Name

```yaml
# ❌ WRONG - Consumer tries to use step output name
producer:
  outputs:
    foo: ${{ steps.generate-foo.outputs.foo1 }}
consumer:
  steps:
  - run: echo "${{ needs.producer.outputs.foo1 }}"  # ❌ Job output is "foo", not "foo1"!
```

```yaml
# ✓ CORRECT - Consumer uses job output name
producer:
  outputs:
    foo: ${{ steps.generate-foo.outputs.foo1 }}
consumer:
  steps:
  - run: echo "${{ needs.producer.outputs.foo }}"   # ✓ Correct job output name
```

### Pitfall 2: Forgetting to Declare Job Output

```yaml
# ❌ WRONG - Step output exists but not exposed as job output
producer:
  # Missing outputs declaration!
  steps:
  - id: generate-foo
    run: echo "foo=bar" >> "$GITHUB_OUTPUT"

consumer:
  needs: producer
  steps:
  - run: echo "${{ needs.producer.outputs.foo }}"  # ❌ Empty! No job output declared
```

```yaml
# ✓ CORRECT - Step output exposed as job output
producer:
  outputs:
    foo: ${{ steps.generate-foo.outputs.foo }}  # ✓ Declared as job output
  steps:
  - id: generate-foo
    run: echo "foo=bar" >> "$GITHUB_OUTPUT"

consumer:
  needs: producer
  steps:
  - run: echo "${{ needs.producer.outputs.foo }}"  # ✓ Works now!
```

---

## Summary

### Three Separate Namespaces:

1. **Shell Variables** (`foo=bar`)
   - Scope: Single run block
   - Access: `$foo`

2. **Step Outputs** (`echo "foo1=bar" >> "$GITHUB_OUTPUT"`)
   - Scope: Steps within same job
   - Access: `${{ steps.step-id.outputs.foo1 }}`

3. **Job Outputs** (`outputs: foo: ${{ ... }}`)
   - Scope: Other jobs
   - Access: `${{ needs.job-name.outputs.foo }}`

### Key Takeaways:

1. **Step output names** and **job output names** are **independent**
2. Job outputs act as a **public interface** between jobs
3. The consumer only sees **job output names**, not step output names
4. You can change internal step output names without breaking consumers, as long as job output names stay the same

### Your Specific Case:

In your file 2:
- Step output is named `foo1` (internal implementation detail)
- Job output is named `foo` (public interface)
- Consumer correctly uses `needs.producer.outputs.foo` (the job output name)
- **This is why it works!** The consumer doesn't care that the step output is named `foo1`