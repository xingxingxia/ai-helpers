---
description: AI-powered merge conflict resolver with semantic analysis
argument-hint: "[--auto] [file-pattern]"
---

## Name
git:resolve-conflicts

## Synopsis
```
/git:resolve-conflicts                    # Interactive mode for all conflicts
/git:resolve-conflicts --auto             # Auto-resolve simple conflicts, pause on complex ones
/git:resolve-conflicts path/to/file.js    # Resolve conflicts in specific file
/git:resolve-conflicts "src/**/*.go"      # Resolve conflicts matching pattern
```

## Description

The `git:resolve-conflicts` command is an AI-powered assistant that helps resolve merge conflicts by analyzing the semantic meaning and intent of conflicting changes from both branches.

**What makes this different from manual conflict resolution:**

1. **Semantic Understanding**: Analyzes what the code DOES, not just what it looks like
2. **Intent Analysis**: Reviews git history to understand WHY changes were made
3. **Intelligent Merging**: Suggests ways to combine both changes when they're complementary
4. **Context-Aware**: Considers the surrounding code and project patterns
5. **Safety First**: Always explains reasoning and requires user approval

**When to use this command:**

- After `git merge` produces conflicts
- After `git rebase` hits conflicts
- After `git cherry-pick` fails with conflicts
- After `git pull` results in merge conflicts
- Any time you see `CONFLICT (content): Merge conflict in <file>`

**What this command does:**

1. Detects all conflicted files in the repository
2. For each conflict, performs deep analysis:
   - Extracts conflicting sections (ours vs theirs)
   - Analyzes git history from both branches to understand intent
   - Examines surrounding code context
   - Identifies if changes are complementary or contradictory
3. Suggests resolution strategies with detailed reasoning
4. Applies approved resolutions
5. Marks files as resolved and prepares for commit

## Implementation

### Phase 1: Conflict Detection and Validation

#### Step 1.1: Check Git State

Verify we're in a conflict state:

```bash
# Check if we're in a merge/rebase/cherry-pick state
git status --porcelain
```

**Expected states:**
- `UU <file>` = Both modified (merge conflict)
- `AA <file>` = Both added (merge conflict)
- `DD <file>` = Both deleted (merge conflict)
- `.git/MERGE_HEAD` exists = In merge state
- `.git/REBASE_HEAD` exists = In rebase state
- `.git/CHERRY_PICK_HEAD` exists = In cherry-pick state

**If NO conflicts detected:**
```
No merge conflicts detected.

Current status: [clean working tree / uncommitted changes / etc.]

This command only works when you have active merge conflicts.
To create conflicts, try: git merge <branch>, git rebase <branch>, or git cherry-pick <commit>
```

Exit gracefully.

**If conflicts ARE detected:**
Proceed to Step 1.2.

#### Step 1.2: Identify Conflicted Files

```bash
# Get list of conflicted files
git diff --name-only --diff-filter=U

# Also check for files marked as unmerged
git ls-files -u | cut -f2 | sort -u
```

**Filter by pattern if provided:**
- If user provided file pattern argument, filter the list to match
- Support glob patterns: `*.go`, `src/**/*.ts`, etc.
- If pattern matches no conflicts, inform user and show all conflicts

**Store for each file:**
- File path
- Conflict type (both modified, both added, etc.)
- File language/type for syntax-aware analysis

#### Step 1.3: Understand the Conflict Context

Determine what operation caused the conflicts:

```bash
# Check the git state
if [ -f .git/MERGE_HEAD ]; then
  conflict_type="merge"
  their_branch=$(git rev-parse --abbrev-ref MERGE_HEAD || git rev-parse --short MERGE_HEAD)
  our_branch=$(git rev-parse --abbrev-ref HEAD)
elif [ -d .git/rebase-merge ] || [ -d .git/rebase-apply ]; then
  conflict_type="rebase"
  their_branch=$(cat .git/rebase-merge/onto 2>/dev/null || echo "unknown")
  our_branch=$(cat .git/rebase-merge/head-name 2>/dev/null | sed 's|refs/heads/||')
elif [ -f .git/CHERRY_PICK_HEAD ]; then
  conflict_type="cherry-pick"
  their_commit=$(git rev-parse --short CHERRY_PICK_HEAD)
  our_branch=$(git rev-parse --abbrev-ref HEAD)
fi
```

Display context to user:
```
═══════════════════════════════════════════════════════════
CONFLICT RESOLUTION SESSION
═══════════════════════════════════════════════════════════

Operation: [merge / rebase / cherry-pick]
Your branch: [our_branch]
Their branch/commit: [their_branch]

Conflicted files found: [N]
[list of files]

═══════════════════════════════════════════════════════════
```

---

### Phase 2: Per-File Conflict Analysis

For each conflicted file, perform the following steps:

#### Step 2.1: Extract Conflict Markers

Parse the file to extract all conflict sections:

```bash
# Read the file and identify conflict markers
# Conflict sections are marked as:
# <<<<<<< HEAD (or ours)
# [our version]
# =======
# [their version]
# >>>>>>> branch-name (or theirs)
```

**For each conflict section in the file, extract:**
1. **Line range**: Where the conflict starts and ends
2. **Our version**: The code between `<<<<<<<` and `=======`
3. **Their version**: The code between `=======` and `>>>>>>>`
4. **Base version** (if available): The common ancestor code
5. **Surrounding context**: 10-20 lines before and after the conflict

#### Step 2.2: Analyze Git History for Intent

For each conflict section, understand the intent behind changes:

**Analyze "our" changes:**
```bash
# Find commits that modified these lines on our branch
git log -p --follow -L <start>,<end>:<file> HEAD --not MERGE_HEAD
# OR for rebase: git log -p --follow -L <start>,<end>:<file> HEAD --not $(cat .git/rebase-merge/onto)
```

**Analyze "their" changes:**
```bash
# Find commits that modified these lines on their branch
git log -p --follow -L <start>,<end>:<file> MERGE_HEAD --not HEAD
# OR for rebase: git log -p --follow -L <start>,<end>:<file> $(cat .git/rebase-merge/onto) --not HEAD^
```

**Extract from git history:**
- Commit messages explaining why changes were made
- Author and date of changes
- Related changes in the same commits
- Patterns across multiple related changes

#### Step 2.3: Semantic Code Analysis

Analyze the actual code changes semantically:

**For our version:**
1. What does this code do functionally?
2. What problem does it solve?
3. Does it add new functionality, fix a bug, refactor, or modify behavior?
4. Are there dependencies on other parts of the codebase?
5. Does it introduce new imports, functions, or variables?

**For their version:**
1. What does this code do functionally?
2. What problem does it solve?
3. Does it add new functionality, fix a bug, refactor, or modify behavior?
4. Are there dependencies on other parts of the codebase?
5. Does it introduce new imports, functions, or variables?

**Compare the two:**
1. **Complementary changes**: Both changes can coexist (e.g., both add different features)
2. **Contradictory changes**: Only one approach can be used (e.g., both fix the same bug differently)
3. **Overlapping changes**: Same area but different aspects (e.g., one refactors, one adds feature)
4. **Duplicate changes**: Both made essentially the same change

#### Step 2.4: Context Analysis

Examine the broader context:

1. **Read surrounding code** (100+ lines before/after)
2. **Check for patterns**: Does the project follow specific conventions that favor one approach?
3. **Analyze imports/dependencies**: Do either version's dependencies conflict with existing code?
4. **Test files**: Are there related test changes that indicate intent?
5. **Similar code elsewhere**: How are similar situations handled in the codebase?

---

### Phase 3: Generate Resolution Strategies

For each conflict, generate 2-4 resolution strategies ranked by confidence:

#### Strategy Types

**1. Accept Ours (Your Changes)**
- **When to suggest**: Their changes conflict with fundamental architecture changes on our branch, or our changes are more recent/comprehensive
- **Confidence indicators**: Our changes touch more of the codebase, newer commits, clearer intent

**2. Accept Theirs (Incoming Changes)**
- **When to suggest**: Their changes fix a bug that our changes don't address, or their approach is more aligned with project patterns
- **Confidence indicators**: Their changes have clearer test coverage, fix known issues, more maintainable

**3. Merge Both (Intelligent Combination)**
- **When to suggest**: Changes are complementary (e.g., our change adds feature A, theirs adds feature B)
- **How to merge**:
  - Identify non-overlapping additions and include both
  - Combine logic carefully (e.g., multiple conditions in if-statement)
  - Preserve both features/fixes
  - Ensure proper ordering and dependencies
- **Confidence indicators**: Clear separation of concerns, no logical contradictions

**4. Custom Hybrid (Best of Both)**
- **When to suggest**: Need to take specific parts from each version
- **How to create**:
  - Take the better error handling from one version
  - Take the better logic from the other
  - Combine the strengths of both approaches
- **Confidence indicators**: Each version has clear advantages in different aspects

**5. Refactor Both (New Solution)**
- **When to suggest**: Both versions have issues, or there's a better way that incorporates the intent of both
- **How to create**: Design a new solution that achieves both intents more elegantly
- **Confidence indicators**: Low - only suggest when both versions seem suboptimal

#### Step 3.1: Present Conflict and Strategies

For each conflict, display:

```
═══════════════════════════════════════════════════════════
FILE: path/to/file.js (Conflict 1 of 3)
═══════════════════════════════════════════════════════════

CONTEXT: Lines 45-67 in src/auth/handler.ts
Function: validateUserToken()

CONFLICT TYPE: Both modified

INTENT ANALYSIS
─────────────────────────────────────────────────────────

OUR CHANGES (branch: feature/add-oauth):
  Commit abc1234 by Alice (2 days ago):
  "Add OAuth2 token validation support"

  What changed: Added OAuth2 bearer token parsing and validation
  Dependencies: New import 'oauth2-lib', new function parseOAuthToken()
  Purpose: Support OAuth2 authentication alongside existing JWT

THEIR CHANGES (branch: main):
  Commit def5678 by Bob (1 day ago):
  "Fix security vulnerability in token validation"

  What changed: Added timing-attack-safe string comparison
  Dependencies: Import 'crypto-safe-compare' utility
  Purpose: Fix CVE-2024-XXXXX timing attack vulnerability

ANALYSIS
─────────────────────────────────────────────────────────

These changes are COMPLEMENTARY:
- Our change adds new OAuth2 functionality
- Their change fixes a security issue in existing code
- Both modifications target the same function
- No logical contradictions detected

Both changes should be preserved.

═══════════════════════════════════════════════════════════
RESOLUTION STRATEGIES
═══════════════════════════════════════════════════════════

STRATEGY #1: Merge Both (Recommended - 95% confidence)
─────────────────────────────────────────────────────────

Combine both changes to preserve OAuth2 support AND security fix.

Resulting code:
\```javascript
function validateUserToken(token) {
  // OAuth2 support from our branch
  if (token.startsWith('Bearer ')) {
    const oauthToken = parseOAuthToken(token);
    return validateOAuthToken(oauthToken);
  }

  // JWT validation with security fix from their branch
  const decoded = jwt.decode(token);
  // Use timing-safe comparison (security fix)
  return cryptoSafeCompare(decoded.signature, expectedSignature);
}
\```

Reasoning:
✓ Preserves OAuth2 functionality from our branch
✓ Includes security fix from their branch
✓ No logical conflicts between the two
✓ Both features work together naturally
✓ Maintains backward compatibility

Dependencies merged:
+ import { parseOAuthToken, validateOAuthToken } from 'oauth2-lib'
+ import { cryptoSafeCompare } from 'crypto-safe-compare'

STRATEGY #2: Accept Theirs, Reapply Ours (85% confidence)
─────────────────────────────────────────────────────────

Start with their security fix, then add our OAuth2 logic on top.

This approach:
✓ Ensures security fix is properly integrated
✓ Reduces risk of breaking their fix
⚠ Requires manual reapplication of OAuth2 logic

STRATEGY #3: Accept Ours, Backport Security Fix (70% confidence)
─────────────────────────────────────────────────────────

Keep our OAuth2 changes and manually add timing-safe comparison.

This approach:
✓ Preserves our implementation exactly
⚠ Might miss related security changes in other parts of their commit
⚠ Higher risk of incomplete security fix

═══════════════════════════════════════════════════════════

What would you like to do?

[1] Use Strategy #1 (Merge Both) - Recommended
[2] Use Strategy #2 (Accept Theirs, Reapply Ours)
[3] Use Strategy #3 (Accept Ours, Backport Security Fix)
[4] Show me the raw diff (see actual conflict markers)
[5] Edit manually (I'll resolve this myself)
[6] Skip this conflict for now
[7] Abort conflict resolution
```

#### Step 3.2: Handle User Selection

**If user selects a strategy [1-3]:**
1. Apply the chosen resolution to the file
2. Show a diff of what was applied
3. Ask for confirmation:
   ```
   Applied Strategy #1. Here's what changed:

   [show diff]

   Does this look correct?
   [y] Yes, mark as resolved and continue
   [n] No, undo and let me choose again
   [e] Edit the result before marking resolved
   ```

**If user selects [4] Show raw diff:**
1. Display the file section with conflict markers
2. Show git diff output
3. Return to strategy selection

**If user selects [5] Edit manually:**
1. Inform user to edit the file manually
2. Offer to open file in editor: `$EDITOR path/to/file`
3. When done, ask if they want to continue with next conflict or review

**If user selects [6] Skip:**
1. Move to next conflict
2. Add to "skipped list" to summarize at the end

**If user selects [7] Abort:**
1. Confirm: "Are you sure? No changes will be saved."
2. If yes: Exit without modifying any files
3. If no: Return to current conflict

---

### Phase 4: Auto-Resolution Mode (--auto flag)

When `--auto` flag is provided, automatically resolve simple conflicts:

#### Auto-Resolution Criteria

**Automatically resolve when ALL conditions met:**
1. **High confidence** (≥90%) in recommended strategy
2. **Low risk** conflict type:
   - Both changes add different code to different parts of a function
   - One side only adds whitespace/formatting
   - One side only adds comments/documentation
   - Changes are in completely different functions in same file
   - One side is empty (simple addition or deletion)
3. **No complex dependencies** introduced
4. **Clear complementary nature** (not contradictory)

**Always pause and ask when:**
1. Confidence < 90%
2. Changes modify the same lines of logic
3. Security-related code detected
4. Both sides modify error handling
5. Test files with contradictory test cases
6. Build configuration files
7. Database migrations or schema changes
8. API contracts or interfaces

#### Auto-Resolution Process

For each conflict:

```
═══════════════════════════════════════════════════════════
AUTO-RESOLVING: src/utils/format.ts (Conflict 2 of 8)
═══════════════════════════════════════════════════════════

Analysis: Both changes add different utility functions
Confidence: 95% (High)
Risk: Low (complementary additions)
Strategy: Merge Both

Auto-applying resolution...
✓ Resolved successfully

[Continue to next conflict]
```

**For conflicts that don't meet auto-resolution criteria:**

```
═══════════════════════════════════════════════════════════
PAUSING AUTO-RESOLUTION: src/auth/handler.ts
═══════════════════════════════════════════════════════════

Reason: Changes modify same logic (confidence: 78%)

[Show full conflict analysis and strategies as in manual mode]
```

---

### Phase 5: Validation and Finalization

#### Step 5.1: Post-Resolution Validation

After resolving each file:

1. **Syntax check**: Verify file has valid syntax
   ```bash
   # For different languages:
   # JavaScript/TypeScript: Use node --check or tsc
   # Python: Use python -m py_compile
   # Go: Use go fmt, gofmt -l
   # Java: Use javac -Xdoclint:none
   ```

2. **Remove conflict markers**: Ensure no `<<<<<<<`, `=======`, `>>>>>>>` remain
   ```bash
   grep -n "^<<<<<<< \|^=======$\|^>>>>>>> " <file>
   ```

3. **Mark as resolved**:
   ```bash
   git add <file>
   ```

4. **Update progress**: Show which conflicts are resolved/remaining

#### Step 5.2: Final Summary

When all conflicts are processed:

```
═══════════════════════════════════════════════════════════
CONFLICT RESOLUTION COMPLETE
═══════════════════════════════════════════════════════════

SUMMARY
─────────────────────────────────────────────────────────

Total conflicts: 8
✓ Auto-resolved: 3
✓ Manually resolved: 4
⊘ Skipped: 1

RESOLVED FILES (marked with git add):
✓ src/utils/format.ts (merged both changes)
✓ src/auth/handler.ts (merged both changes)
✓ src/api/routes.ts (accepted theirs)
✓ tests/auth.test.ts (custom hybrid)
✓ src/config/settings.ts (merged both)
✓ src/models/user.ts (accepted ours)
✓ package.json (merged both)

SKIPPED FILES (still conflicted):
⊘ database/migrations/0042_alter_schema.sql
  Reason: User chose to skip
  You'll need to resolve this manually

NEXT STEPS
─────────────────────────────────────────────────────────

[If all conflicts resolved:]

All conflicts have been resolved and staged!

You can now:
1. Review the changes: git diff --cached
2. Continue the [merge/rebase/cherry-pick]:
   [If merge:     git commit]
   [If rebase:    git rebase --continue]
   [If cherry-pick: git cherry-pick --continue]
3. Or abort: git [merge/rebase/cherry-pick] --abort

Would you like me to:
[1] Show git diff --cached (review all changes)
[2] Continue the [operation] (run git [merge/rebase/cherry-pick] --continue)
[3] Commit the merge (run git commit with suggested message)
[4] Exit (you'll finish manually)

[If some conflicts skipped:]

⚠ Warning: 1 file still has unresolved conflicts

Before you can continue, you must resolve:
- database/migrations/0042_alter_schema.sql

Options:
[1] Resolve remaining conflicts now (re-run /git:resolve-conflicts)
[2] Open skipped file in editor
[3] Show conflict markers for skipped file
[4] Exit (you'll resolve manually)

═══════════════════════════════════════════════════════════
```

#### Step 5.3: Continue the Git Operation (Optional)

If user chooses to continue the git operation automatically:

**For merge:**
```bash
git commit -m "Merge branch '<their-branch>' into <our-branch>

Resolved conflicts in:
- <file1>: <strategy used>
- <file2>: <strategy used>

Generated with AI assistant"
```

**For rebase:**
```bash
git rebase --continue
```

**For cherry-pick:**
```bash
git cherry-pick --continue
```

---

### Error Handling

#### Common Errors and Solutions

| Error | Solution |
|-------|----------|
| No conflicts detected | Explain current state, suggest what to do |
| File modified during resolution | Detect external changes, ask user to reload or continue |
| Syntax error after resolution | Show error, offer to undo the resolution |
| Git operation fails (rebase/merge --continue) | Show git error, suggest manual intervention |
| Pattern matches no files | Show all conflicts, ask if they want to proceed with all |

#### Safety Checks

**Before applying any resolution:**
1. Check if file has been modified externally since analysis started
2. Verify the conflict markers still exist in expected locations
3. Ensure git is still in conflict state

**After applying resolution:**
1. Verify file has valid syntax
2. Verify no conflict markers remain
3. Verify file was successfully staged with `git add`

## Examples

### Example 1: Interactive Resolution of All Conflicts

```
/git:resolve-conflicts
```

**Output:**
```
═══════════════════════════════════════════════════════════
CONFLICT RESOLUTION SESSION
═══════════════════════════════════════════════════════════

Operation: merge
Your branch: feature/user-auth
Their branch: main

Conflicted files found: 3
- src/auth/handler.ts
- src/models/user.ts
- package.json

═══════════════════════════════════════════════════════════

[Proceeds to analyze and resolve each conflict interactively]
```

### Example 2: Auto-Resolve Simple Conflicts

```
/git:resolve-conflicts --auto
```

**Output:**
```
═══════════════════════════════════════════════════════════
AUTO-RESOLUTION MODE
═══════════════════════════════════════════════════════════

Auto-resolving simple conflicts (will pause on complex ones)

✓ src/utils/format.ts - Auto-resolved (95% confidence)
✓ package.json - Auto-resolved (92% confidence)

⏸ Pausing on: src/auth/handler.ts
  Reason: Changes modify same logic (78% confidence)

[Shows full analysis and strategies for manual decision]
```

### Example 3: Resolve Specific File

```
/git:resolve-conflicts src/auth/handler.ts
```

**Output:**
```
═══════════════════════════════════════════════════════════
RESOLVING: src/auth/handler.ts
═══════════════════════════════════════════════════════════

Found 2 conflicts in this file

[Analyzes and resolves each conflict in the specified file]
```

### Example 4: Resolve Multiple Files by Pattern

```
/git:resolve-conflicts "src/**/*.ts"
```

**Output:**
```
═══════════════════════════════════════════════════════════
PATTERN MATCH: src/**/*.ts
═══════════════════════════════════════════════════════════

Matching conflicted files: 2
- src/auth/handler.ts
- src/models/user.ts

Non-matching conflicts skipped: 1
- package.json

[Proceeds to resolve only the matching files]
```

### Example 5: During Rebase

```
$ git rebase main
CONFLICT (content): Merge conflict in src/api.ts
error: could not apply abc1234... Add rate limiting

$ /git:resolve-conflicts
```

**Output:**
```
═══════════════════════════════════════════════════════════
CONFLICT RESOLUTION SESSION
═══════════════════════════════════════════════════════════

Operation: rebase
Your branch: feature/rate-limit
Rebasing onto: main

Conflicted files found: 1
- src/api.ts

[Analyzes conflict and provides resolution strategies]

After resolution:

Would you like me to:
[1] Show git diff --cached
[2] Continue the rebase (run git rebase --continue)
[3] Exit
```

### Example 6: Complex Security-Related Conflict

```
/git:resolve-conflicts --auto
```

**Output:**
```
✓ Auto-resolved: src/utils/logger.ts

⏸ Pausing on: src/auth/crypto.ts
  Reason: Security-related code detected

═══════════════════════════════════════════════════════════
FILE: src/auth/crypto.ts (Conflict 1 of 1)
═══════════════════════════════════════════════════════════

⚠ SECURITY ALERT: This file handles cryptographic operations

INTENT ANALYSIS
─────────────────────────────────────────────────────────

OUR CHANGES:
  "Upgrade encryption algorithm to AES-256-GCM"

THEIR CHANGES:
  "Fix IV reuse vulnerability (CVE-2024-XXXXX)"

ANALYSIS: CONTRADICTORY CHANGES
Both changes modify the encryption function differently.
⚠ Both address security concerns - expert review recommended.

STRATEGY #1: Accept Theirs + Reapply Ours (65% confidence)
⚠ Lower confidence due to security implications
⚠ Manual review strongly recommended

[Detailed analysis continues...]

Given the security-sensitive nature, would you like to:
[1] Proceed with suggested strategy
[2] Mark for manual review
[3] Consult documentation
```

## Return Value

**Success Cases:**

1. **All conflicts resolved:**
   - Summary of files resolved
   - Strategies used for each file
   - Next steps to continue git operation
   - Option to auto-continue merge/rebase/cherry-pick

2. **Partial resolution:**
   - Summary of resolved files
   - List of skipped files
   - Instructions for resolving remaining conflicts

3. **Auto-resolution with pauses:**
   - Count of auto-resolved conflicts
   - Interactive resolution for complex conflicts
   - Combined summary at the end

**Informational Cases:**

- No conflicts detected (with explanation of current state)
- No conflicts matching pattern (with list of all conflicts)

**Error Cases:**

- Invalid git state
- File access errors
- Syntax validation failures after resolution

## Arguments

- **--auto** (flag): Enable auto-resolution mode for simple conflicts
  - Auto-resolves high-confidence (≥90%), low-risk conflicts
  - Pauses on complex or security-related conflicts
  - Shows progress for each auto-resolved conflict

- **file-pattern** (optional): Resolve conflicts only in files matching pattern
  - Can be exact path: `src/auth/handler.ts`
  - Can be glob pattern: `src/**/*.ts`, `*.go`, `tests/**/*`
  - If no conflicts match, shows all conflicts and offers to proceed

**Note**: Arguments can be combined: `/git:resolve-conflicts --auto "src/**/*.ts"`

## Notes

- **Safety First**: All resolutions require user approval unless in `--auto` mode with high confidence
- **Semantic Understanding**: Analyzes code meaning, not just text differences
- **Intent-Driven**: Reviews git history to understand why changes were made
- **Context-Aware**: Considers surrounding code and project patterns
- **Syntax Validation**: Checks that resolved files have valid syntax
- **Comprehensive**: Handles merge, rebase, and cherry-pick conflicts
- **Educational**: Explains reasoning for each suggested resolution
- **Flexible**: Supports manual editing at any point
- **Complementary**: Can be used alongside `/git:fix-cherrypick-robot-pr` for robot PR failures

## See Also

- `/git:fix-cherrypick-robot-pr` - Fix failed cherrypick-robot PRs (may use this command for conflict resolution)
- `/git:cherry-pick-by-patch` - Alternative cherry-pick method that may avoid some conflicts
- `git status` - Check current conflict state
- `git diff` - View differences between versions
- `git log` - Review commit history for intent analysis
