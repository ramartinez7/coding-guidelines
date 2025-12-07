# Collaboration Quick Start Guide

> **TL;DR**: One pattern per PR, draft PR early, update `__index__.md` last.

## The 5-Minute Guide to Avoiding Conflicts

### 1. Before You Start

```bash
# Check what others are working on
# Look at: https://github.com/ramartinez7/coding-guidelines/pulls
# If someone is already working on your pattern, coordinate with them
```

### 2. Claim Your Work

```bash
# Create your branch
git checkout -b feature/yourname/pattern-name

# Make your pattern file
# ... edit csharp/patterns/your-pattern.md ...

# Open a DRAFT PR immediately (even if not done)
git add csharp/patterns/your-pattern.md
git commit -m "WIP: Add [pattern name]"
git push -u origin feature/yourname/pattern-name
# Then create draft PR on GitHub
```

**Why?** This signals to everyone else that you're working on this pattern.

### 3. The Golden Rule

**One Pattern = One PR**

✅ Good:
- PR #1: Add circuit-breaker.md
- PR #2: Add retry-pattern.md
- PR #3: Add rate-limiting.md

❌ Bad:
- PR #1: Add circuit-breaker.md + retry-pattern.md + rate-limiting.md

**Why?** Multiple patterns = more conflicts + slower reviews

### 4. The __index__.md Problem

The `__index__.md` file is the highest conflict zone. Everyone wants to update it.

**Solution: Update it LAST**

```bash
# Step 1: Write your pattern completely
# ... work on your-pattern.md ...

# Step 2: Commit your pattern (without index)
git add csharp/patterns/your-pattern.md
git commit -m "Add your pattern"

# Step 3: Rebase to get latest changes
git checkout main
git pull
git checkout feature/yourname/pattern-name
git rebase main

# Step 4: NOW update __index__.md
# ... add your one entry ...
git add csharp/patterns/__index__.md
git commit -m "Add your-pattern to index"

# Step 5: Push and mark ready for review
git push --force-with-lease
```

### 5. Rebase Often

```bash
# At least once a week, or when you see main updated
git checkout main
git pull origin main
git checkout feature/yourname/pattern-name
git rebase main
# Fix any conflicts
git push --force-with-lease
```

**Why?** Small, frequent conflict resolution is easier than one huge conflict at the end.

## Quick Reference

| Situation | What To Do |
|-----------|------------|
| Starting new pattern | Check open PRs, create branch, open draft PR |
| Someone else is working on same pattern | Coordinate in PR comments or issues |
| Multiple patterns to add | Create separate PRs for each |
| Need to update __index__.md | Do it last, after rebasing |
| PR has conflicts | Rebase on main, resolve, force-push |
| Want to improve existing pattern | Create separate PR, use `fix/` branch name |

## Conflict-Free Workflow Checklist

- [ ] Checked open PRs for duplicate work
- [ ] Created descriptive branch name: `feature/name/pattern`
- [ ] Opened draft PR early to claim work
- [ ] Keeping PR focused (one pattern only)
- [ ] Pattern file complete and committed
- [ ] Rebased on latest main
- [ ] Updated `__index__.md` (if adding pattern)
- [ ] Marked PR ready for review

## Common Questions

**Q: What if I want to add 5 related patterns?**
A: Create 5 separate PRs. They can be reviewed and merged independently.

**Q: Someone else updated `__index__.md` while I was working. Now I have conflicts!**
A: Rebase on main, then carefully merge just your entry into the updated index.

**Q: Can I update multiple existing patterns in one PR?**
A: Only if they're closely related (e.g., fixing the same typo across patterns). Otherwise, separate PRs.

**Q: How long should I wait for PR review?**
A: Minor fixes: 1-2 days. New patterns: 3-5 days. Ping maintainer if no response after a week.

**Q: Should I update documentation and patterns in the same PR?**
A: If the documentation is directly related to your pattern (e.g., adding it to philosophy.md), yes. Otherwise, separate PR.

## Branch Naming

Use this formula:
```
<type>/<your-name>/<description>

Types:
  feature/  - New patterns
  fix/      - Corrections
  docs/     - Documentation (non-pattern)
```

Examples:
- `feature/alex/circuit-breaker`
- `fix/jordan/typo-in-honest-functions`
- `docs/sam/improve-contributing-guide`

## Files by Conflict Risk

| File | Risk | Strategy |
|------|------|----------|
| `patterns/*.md` (individual) | 🟢 Low | One person per file |
| `patterns/__index__.md` | 🔴 High | Update last, keep minimal |
| `philosophy.md` | 🟡 Medium | Discuss changes first |
| `style.md` | 🟡 Medium | Discuss changes first |

## When In Doubt

Read the full guide: [CONTRIBUTING.md](../CONTRIBUTING.md)

Or ask in your PR or create an issue!

---

**Remember**: The goal is to make collaboration smooth. When in doubt, smaller PRs and more communication = fewer conflicts. 🎯
