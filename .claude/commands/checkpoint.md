# Checkpoint Command

Create or update checkpoint for current day's work.

---

## Workflow

You will follow these steps to create a comprehensive checkpoint:

### Step 1: Identify Context
- Determine current day number (1-30)
- Identify ETAPA and items being worked on
- Reference PROJECT_SPECS.md for requirements

### Step 2: Gather Metrics
Run these commands to collect data:

```bash
# Test results
python manage.py test

# Coverage
coverage run --source='.' manage.py test
coverage report

# Type checking
mypy apps/ src/

# Linting
flake8 apps/ src/
```

### Step 3: Analyze Git Status
```bash
# See what changed today
git diff main --stat

# Recent commits
git log --oneline --since="1 day ago"
```

### Step 4: Use checkpoint-manager Agent
```bash
> Use checkpoint-manager to create checkpoint for Day X
```

Provide the agent with:
- Completed items list
- Blockers encountered
- Test metrics
- Coverage percentage
- Key learnings
- Next session plan

### Step 5: Update Context Session
If working on a feature with context session file:
```bash
> Update .claude/sessions/context_session_{feature_name}.md with today's progress
```

### Step 6: Commit Checkpoint
```bash
git add checkpoints/checkpoint_day_X.md
git commit -m "checkpoint: Day X - [brief summary]"
```

---

## Template Reference

Checkpoint file structure:
```markdown
# Checkpoint: Day X - [Feature Name]

## Date
YYYY-MM-DD

## Session Duration
[Hours worked]

## Completed Items
- [X] Item 1
- [X] Item 2

## Work in Progress
- [ ] Item 3 (60% complete)

## Blockers Encountered
### Blocker 1
**Problem**: [Description]
**Solution**: [How resolved or plan]

## Key Learnings
- Learning 1
- Learning 2

## Code Metrics
- Tests: X passing, Y failing
- Coverage: Z%
- Type Check: PASS/FAIL
- Security: PASS/FAIL

## Technical Debt
- [Item to fix later]

## Next Session Plan
1. [Specific task]
2. [Specific task]
```

---

## When to Use

- **End of each work session** (recommended)
- **Before long break** (lunch, end of day)
- **After completing major milestone**
- **When switching between features**

---

## Benefits

- Preserves context for next session
- Tracks progress over 30 days
- Documents blockers and solutions
- Provides audit trail
- Helps with weekly retrospectives
