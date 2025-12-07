# Contributing to Coding Guidelines

Welcome! This guide will help you contribute to the C# coding patterns documentation while avoiding merge conflicts and maintaining consistency.

## Table of Contents

- [Quick Start](#quick-start)
- [Repository Structure](#repository-structure)
- [Collaboration Strategy](#collaboration-strategy)
- [File Ownership & Conflict Prevention](#file-ownership--conflict-prevention)
- [Branching Strategy](#branching-strategy)
- [Creating New Patterns](#creating-new-patterns)
- [Editing Existing Patterns](#editing-existing-patterns)
- [Pull Request Process](#pull-request-process)
- [Style Guidelines](#style-guidelines)
- [Pattern Template](#pattern-template)

---

## Quick Start

1. **Before starting work**, check [open PRs](../../pulls) to see if someone is already working on the same pattern or file
2. **Claim your work** by opening a draft PR immediately with the pattern name in the title
3. **Work in your branch** using the naming convention: `feature/<your-name>/<pattern-name>`
4. **Keep PRs small** - one pattern per PR is ideal
5. **Update __index__.md last** to minimize conflicts

---

## Repository Structure

```
/csharp/
  philosophy.md           # Core philosophies (edited rarely)
  style.md                # C# coding style conventions (edited rarely)
  /patterns/
    __index__.md          # Catalog of all patterns (HIGH CONFLICT ZONE)
    pattern-name.md       # Individual pattern files (LOW CONFLICT)
```

**Conflict Zones:**
- **HIGH**: `__index__.md` - Many contributors will touch this file
- **MEDIUM**: `philosophy.md`, `style.md` - Occasionally updated
- **LOW**: Individual pattern files - Usually one person per file

---

## Collaboration Strategy

### The Golden Rules

1. **One Pattern = One PR** - This is the single most important rule for avoiding conflicts
2. **Draft PRs Early** - Create a draft PR as soon as you start to signal your work to others
3. **Small, Focused Changes** - Easier to review and less likely to conflict
4. **Communicate in Issues** - Discuss major changes before implementing
5. **Rebase Often** - Keep your branch up to date with `main`

### Pattern Ownership Model

To prevent conflicts, we use a **temporary ownership** model:

| Phase | Status | Rule |
|-------|--------|------|
| **Planning** | Create Issue | State which pattern you want to work on |
| **Claimed** | Open Draft PR | You "own" this pattern until PR is merged |
| **In Review** | Mark Ready for Review | Others can suggest changes via review |
| **Released** | PR Merged | Pattern is available for others to improve |

**Important**: If you're inactive on a pattern for >7 days without updates, others may take over the work.

---

## File Ownership & Conflict Prevention

### Strategy 1: Work on Different Files

The easiest way to avoid conflicts is to **work on completely different files**.

✅ **Safe - No Conflicts:**
- Alice works on `lazy-initialization.md`
- Bob works on `circuit-breaker.md`
- Carol works on `rate-limiting.md`

### Strategy 2: Pattern Categories

If multiple people want to add patterns simultaneously, **divide by category**:

| Category | Examples | Typical Owner |
|----------|----------|---------------|
| **Security** | Authentication, Authorization, Secrets | Security specialists |
| **Performance** | Memory, Allocation, Caching | Performance engineers |
| **API Design** | Versioning, Content negotiation | API team |
| **DDD** | Aggregates, Repositories, Events | Domain experts |

Assign one person per category to batch-add multiple patterns in that area.

### Strategy 3: The __index__.md Problem

`__index__.md` is the highest-conflict file. To manage it:

**Option A: Designate a Maintainer**
- One person (the maintainer) handles all `__index__.md` updates
- Contributors submit patterns without updating `__index__.md`
- Maintainer adds entries in a separate PR after merging

**Option B: Update Last**
- Add your pattern file first
- Only update `__index__.md` right before marking PR ready for review
- Rebase immediately before updating to get latest changes
- Keep the change minimal (just your entry)

**Option C: Batch Updates**
- Multiple contributors add patterns without touching `__index__.md`
- Maintainer does one batch update weekly adding all new patterns

**Recommendation**: Use Option A or C for teams >3 contributors.

### Strategy 4: Coordinate via Issues

Before starting work:

1. Check [open issues](../../issues) for your topic
2. Comment on related issues: "I'm working on this pattern"
3. If no issue exists, create one: "Add pattern: [Pattern Name]"
4. Link your PR to the issue

### Strategy 5: Use Conventional Branch Names

Use this naming pattern:
```
feature/<your-name>/<pattern-name>
```

Examples:
- `feature/alice/circuit-breaker`
- `feature/bob/rate-limiting`
- `feature/carol/saga-pattern`

This makes it immediately clear who is working on what.

---

## Branching Strategy

### Branch Naming

```
feature/<name>/<description>     # New patterns or major additions
fix/<name>/<description>         # Corrections to existing patterns
docs/<name>/<description>        # Documentation improvements (non-pattern)
```

Examples:
- `feature/jordan/retry-pattern`
- `fix/sam/fix-typo-in-honest-functions`
- `docs/alex/improve-getting-started`

### Workflow

```bash
# 1. Start from main
git checkout main
git pull origin main

# 2. Create your feature branch
git checkout -b feature/yourname/pattern-name

# 3. Make your changes
# (edit files)

# 4. Commit with descriptive messages
git add .
git commit -m "Add circuit breaker pattern"

# 5. Push and open Draft PR immediately
git push -u origin feature/yourname/pattern-name

# 6. Continue working, commit frequently
git add .
git commit -m "Add code examples to circuit breaker"
git push

# 7. Rebase before finishing (to get latest changes)
git checkout main
git pull origin main
git checkout feature/yourname/pattern-name
git rebase main
# (resolve any conflicts)
git push --force-with-lease

# 8. Mark PR as ready for review
```

### Keeping Your Branch Updated

Rebase frequently to avoid large conflicts later:

```bash
# Weekly (or when you see main has updates)
git checkout main
git pull origin main
git checkout feature/yourname/pattern-name
git rebase main
git push --force-with-lease
```

---

## Creating New Patterns

### Step-by-Step Process

1. **Check for Duplicates**
   - Search existing patterns in `__index__.md`
   - Check open PRs and issues
   
2. **Create an Issue** (optional but recommended)
   - Title: "Add pattern: [Pattern Name]"
   - Description: Brief explanation of the pattern
   
3. **Create Your Branch**
   ```bash
   git checkout -b feature/yourname/pattern-name
   ```

4. **Create the Pattern File**
   - Use kebab-case: `my-pattern-name.md`
   - Place in `/csharp/patterns/`
   - Use the [Pattern Template](#pattern-template) below

5. **Open a Draft PR Immediately**
   - Title: "Add pattern: [Pattern Name]"
   - Mark as draft
   - This signals to others you're working on it

6. **Write the Pattern**
   - Follow the template structure
   - Include code examples (❌ before, ✅ after)
   - Add links to related patterns

7. **Update __index__.md** (do this LAST)
   - Rebase to get latest changes first
   - Add ONE entry in the appropriate category
   - Keep the change minimal

8. **Mark PR Ready for Review**

### Dos and Don'ts

✅ **Do:**
- Create draft PR early
- Work on one pattern at a time
- Follow the established template
- Link to related patterns
- Use descriptive commit messages

❌ **Don't:**
- Work on multiple patterns in one PR
- Update `__index__.md` early in your work
- Start work without checking for duplicates
- Leave PRs stale for weeks

---

## Editing Existing Patterns

Improvements to existing patterns are welcome! Follow these guidelines:

### Minor Changes (No Conflict Risk)

For typos, grammar, or small clarifications:

1. Create branch: `fix/yourname/fix-pattern-name`
2. Make the change
3. Open PR with title: "Fix: [brief description] in [pattern name]"
4. Small PRs get fast reviews!

### Major Changes (Restructuring, New Examples)

For significant rewrites or additions:

1. **Open an issue first** to discuss the change
2. Wait for maintainer feedback
3. Once approved, follow the same process as minor changes
4. Expect more detailed review

### Adding Related Patterns Section

If you're adding cross-references to multiple patterns:

1. **Do it in a separate PR** from your new pattern
2. Title: "Add cross-references to [pattern name]"
3. This keeps your new pattern PR clean and focused

---

## Pull Request Process

### PR Checklist

Before marking your PR as ready for review:

- [ ] Pattern file follows the template structure
- [ ] Code examples are correct and compile conceptually
- [ ] Cross-references to related patterns are included
- [ ] `__index__.md` is updated (if adding new pattern)
- [ ] No unrelated changes included
- [ ] Branch is rebased on latest main
- [ ] Commit messages are clear and descriptive
- [ ] PR description explains what and why

### PR Title Format

```
Add pattern: [Pattern Name]
Fix: [Issue] in [Pattern Name]
Update: [Pattern Name] - [Brief Description]
```

### PR Description Template

```markdown
## What

Brief description of the pattern or change.

## Why

Why this pattern is useful / why this change is needed.

## Related

- Closes #[issue number]
- Related to #[issue number]
- Depends on #[PR number]

## Checklist

- [ ] Follows pattern template
- [ ] Code examples included
- [ ] Cross-references added
- [ ] __index__.md updated
```

### Review Process

1. **Draft PR** - Work in progress, get early feedback
2. **Ready for Review** - Complete, waiting for maintainer
3. **Changes Requested** - Address feedback
4. **Approved** - Will be merged soon
5. **Merged** - Live on main branch

**Expected Timeline:**
- Minor fixes: 1-2 days
- New patterns: 3-5 days
- Major changes: 1-2 weeks

---

## Style Guidelines

### File Naming

- Use **kebab-case**: `my-pattern-name.md`
- Be descriptive: `lazy-initialization.md` not `lazy.md`
- Match the pattern title (lowercase, hyphens for spaces)

### Writing Style

- **Educational but pragmatic** - Focus on practical use
- **Clear examples** - Show before (❌) and after (✅)
- **Explain the "why"** - Don't just show the "how"
- **Link related patterns** - Help readers discover connections

### Code Examples

```csharp
// ❌ Before: What NOT to do
public void BadExample()
{
    // Show the problem
}

// ✅ After: The improved approach
public void GoodExample()
{
    // Show the solution
}
```

**Code Style:**
- Use C# 12+ syntax
- Prefer `record` types for value objects
- Use pattern matching where appropriate
- Show immutable designs
- Properly annotate nullable reference types

### Cross-References

Link to related patterns using relative paths:

```markdown
See also:
- [Honest Functions](./honest-functions.md)
- [Value Semantics](./value-semantics.md)
```

---

## Pattern Template

Use this template when creating a new pattern:

```markdown
# Pattern Name

> One-sentence summary of what this pattern does.

## Problem

Describe the problem or code smell this pattern addresses.

**Example of the Problem:**

```csharp
// ❌ Code demonstrating the problem
public class BadExample
{
    // Show what's wrong
}
```

**Issues:**
- Specific problem #1
- Specific problem #2
- Why this is problematic

---

## Solution

Describe the solution at a high level.

### Implementation

```csharp
// ✅ Improved implementation
public class GoodExample
{
    // Show the solution
}
```

### Key Points

- Important aspect of the solution
- Another key point
- Why this works better

---

## Benefits

- **Benefit 1**: Explanation
- **Benefit 2**: Explanation
- **Benefit 3**: Explanation

---

## Trade-offs

- **Cost**: What do you give up?
- **Complexity**: Is it more code?
- **When to use**: Appropriate scenarios
- **When to avoid**: Inappropriate scenarios

---

## Real-World Example

Show a concrete example from a realistic scenario.

```csharp
// Real-world code example
```

---

## Related Patterns

- [Related Pattern 1](./related-pattern-1.md) - How it relates
- [Related Pattern 2](./related-pattern-2.md) - How it relates

---

## References

- Link to external resources (if applicable)
- Books, articles, or documentation
```

---

## Questions or Issues?

- **Found a bug in a pattern?** Open an issue
- **Have a question?** Start a discussion
- **Want to propose a new pattern?** Open an issue first
- **Need help contributing?** Tag a maintainer in your PR

---

## Thank You!

Every contribution makes this resource better for the entire C# community. We appreciate your time and expertise!
