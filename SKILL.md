---
name: code-augur
version: 2.1.0
description: AI-powered code pattern predictor and optimizer for TypeScript/JavaScript/Go projects
author: SMOUJBOT
tags: [prediction, generation, optimization, patterns, code]
dependencies:
  - node: >=18
  - python: >=3.9
  - ast-parser: ^2.0.0
  - pattern-matcher: ^1.3.0
  - openai-api: >=4.0.0
required_env:
  - OPENAI_API_KEY
  - CODE_AUGUR_MODEL (default: gpt-4-turbo-preview)
  - CODE_AUGUR_TEMPERATURE (default: 0.2)
---

# Code Augur

Intelligent code pattern prediction and optimization engine that analyzes existing codebases to predict next implementations, suggest architectural improvements, and generate optimized solutions based on established project patterns.

## Purpose

Real-world use cases:
- **API Endpoint Prediction**: When you start a new REST endpoint, Code Augur predicts the complete controller/service/repository stack based on existing patterns in the project.
- **Testing Pattern Generation**: Automatically generates test file structure and boilerplate matching the project's existing testing framework (Jest, Vitest, Go test, etc.).
- **Migration Pattern Detection**: Identifies when code is manually implementing patterns that have library equivalents (e.g., manual debouncing when lodash.debounce is already a dependency).
- **Performance Anti-Pattern Discovery**: Scans for N+1 queries, unnecessary re-renders in React, inefficient loops, and suggests optimized alternatives used elsewhere in the codebase.
- **Consistency Enforcement**: Detects when similar components/functions use different patterns and suggests standardization to the dominant approach.
- **Boilerplate Generation**: Creates new modules following exact project conventions: error handling, logging, validation, type definitions, and exports.

## Scope

Commands that can be executed:
- `code-augur analyze [path] [--extensions] [--exclude] [--depth]`
- `code-augur predict [--file] [--line] [--context] [--output]`
- `code-augur optimize [--target] [--strategy] [--dry-run]`
- `code-augur suggest [--issue-type] [--module] [--format]`
- `code-augur validate [--pattern] [--test] [--coverage]`
- `code-augur learn [--examples] [--categories] [--weight]`
- `code-augur export [--format json|yaml|md] [--include-stats]`

## Work Process

Detailed execution steps:

1. **Environment Validation**
   ```bash
   # Verify dependencies and API access
   node -e "console.log(require('ast-parser').version)"
   python3 -c "import pattern_matcher; print(pattern_matcher.__version__)"
   test -n "$OPENAI_API_KEY" || { echo "Missing OPENAI_API_KEY"; exit 1; }
   ```

2. **Codebase Scanning Phase**
   ```bash
   # Default: analyzes src/, lib/, pkg/ with language auto-detection
   code-augur analyze . \
     --extensions ts,tsx,js,jsx,go \
     --exclude node_modules,dist,.git,coverage \
     --depth 3  # follow imports up to 3 levels
   ```
   Output: `analysis-report.json` containing:
   - Detected AST patterns per language
   - Frequency of error handling approaches
   - Identified architectural layers
   - Common utility libraries in use
   - Test file patterns (Vitest/Jest setup)

3. **Pattern Extraction & Indexing**
   - Parse each file into AST
   - Extract function signatures, error handling patterns, async patterns, type definitions
   - Cluster similar patterns using similarity scoring
   - Store in local SQLite cache: `~/.code-augur/pattern-cache.db`

4. **Prediction Generation**
   For `predict` command:
   ```bash
   # User is writing a new controller in Express.js, wants pattern match
   code-augur predict \
     --file src/controllers/UserController.ts \
     --line 15  # cursor position where new function should be inserted \
     --context "POST endpoint for user registration with validation" \
     --output generated-snippet.ts
   ```
   Process:
   - Fetch all existing controller methods from analysis cache
   - Match based on: HTTP method, parameter patterns, validation approach, response format, error handling
   - Generate 3 candidate implementations with confidence scores
   - Select highest confidence (>0.85) or present options

5. **Optimization Analysis**
   For `optimize` command:
   ```bash
   code-augur optimize \
     --target src/utils/dataProcessor.ts \
     --strategy performance \
     --dry-run  # shows diffs without writing
   ```
   Steps:
   - Identify loops that could use .map/.filter instead of forEach
   - Find object/array operations with O(n²) complexity
   - Detect redundant API calls that could be cached
   - Compare against similar functions that already have optimizations
   - Generate optimization diff with explanation

6. **Validation & Testing**
   ```bash
   code-augur validate \
     --pattern "async-await-without-try-catch" \
     --test "detects patterns in isolation mode" \
     --coverage 0.8  # require 80% pattern confidence
   ```
   - Run pattern detection against known-good code samples
   - Verify no false positives on clean code
   - Measure precision/recall of pattern matcher

7. **Export & Integration**
   ```bash
   code-augur export --format md --include-stats > PATTERNS.md
   ```
   Creates documentation of all detected patterns for team reference.

## Golden Rules

1. **Never modify production code without --dry-run first**. Always review generated changes and run tests.
2. **Cache invalidation**: When dependencies change, run `code-augur analyze --rebuild-cache` to avoid stale predictions.
3. **Confidence threshold**: Ignore predictions with confidence < 0.75 unless explicitly overridden with `--force`.
4. **Pattern drift detection**: Weekly, run `code-augur validate --drift-check` to ensure predictions still match evolving codebase.
5. **Secrets safety**: Never send code containing API keys, passwords, or tokens to prediction engine. Use `--sanitize-secrets` (enabled by default) to replace with placeholders.
6. **Performance budget**: If analysis takes >30 seconds, reduce `--depth` or `--extensions` scope.
7. **Human review mandatory** for any optimization that touches error handling, security checks, or data validation.
8. **Version lock**: Cache is tied to project's `package.json` and `go.mod` hash. If dependencies shift significantly, wipe cache at `~/.code-augur/pattern-cache.db`.

## Examples

### Example 1: Predict a new React component pattern
```bash
# Existing: Button, Input, Modal components use a factory pattern with forwardRef
code-augur predict \
  --file src/components/Select.tsx \
  --line 8 \
  --context "Create a select dropdown component with options prop and onChange handler" \
  --output Select.generated.tsx
```

**Generated output** (Select.generated.tsx):
```tsx
import React, { forwardRef } from 'react';
import { BaseComponent } from './BaseComponent';

interface SelectProps {
  options: Array<{ value: string; label: string }>;
  value?: string;
  onChange?: (value: string) => void;
  disabled?: boolean;
}

export const Select = forwardRef<HTMLSelectElement, SelectProps>((props, ref) => {
  const { options, value, onChange, disabled = false } = props;

  return (
    <BaseComponent>
      <select
        ref={ref}
        value={value}
        onChange={(e) => onChange?.(e.target.value)}
        disabled={disabled}
        className="select-base"
      >
        {options.map(opt => (
          <option key={opt.value} value={opt.value}>
            {opt.label}
          </option>
        ))}
      </select>
    </BaseComponent>
  );
});
```
Confidence: 0.91 (matches Button/Input/Modal factory pattern)

### Example 2: Optimize a Go function
```bash
code-augur optimize --target pkg/service/user_service.go --strategy memory
```

**Before**:
```go
func (s *Service) GetUserRoles(userID string) ([]string, error) {
    roles := []string{}
    user, err := s.repo.FindByID(userID)
    if err != nil {
        return nil, err
    }
    for _, r := range user.Roles {
        role, err := s.roleRepo.Get(r.ID)
        if err != nil {
            return nil, err
        }
        roles = append(roles, role.Name)
    }
    return roles, nil
}
```

**Optimized** (detected that another service uses eager loading):
```go
func (s *Service) GetUserRoles(userID string) ([]string, error) {
    // Pattern matched: similar to OrderService.GetOrderItems using Preload
    user, err := s.repo.FindWithRoles(userID) // uses LEFT JOIN to preload
    if err != nil {
        return nil, err
    }
    roles := make([]string, 0, len(user.Roles))
    for _, r := range user.Roles {
        roles = append(roles, r.Name)
    }
    return roles, nil
}
```

### Example 3: Detect migration opportunities
```bash
code-augur suggest --issue-type "deprecated-pattern" --module src/auth/
```

**Output**:
```
File: src/auth/session.ts
Line 54: Manual JWT token refresh with setTimeout
→ Detected 4 similar occurrences across codebase
→ Project has 'jsonwebtoken' v9.8.0 which supports auto-refresh
→ Suggestion: Replace custom refresh logic with jwt.verify(options.expiresIn)
→ Confidence: 0.94
→ Migration effort: 2 hours (replace 4 functions)
```

## Rollback Commands

Specific rollback procedures:

1. **Undo prediction generation** (before commit):
   ```bash
   # If you ran predict with --output but haven't committed
   rm generated-snippet.ts
   # Or if you used --in-place flag:
   git checkout -- src/controllers/UserController.ts
   ```

2. **Restore from optimization backup**:
   Code Augur automatically creates `file.optimized-backup-<timestamp>.ts` when using `--in-place`.
   ```bash
   code-augur restore --backup-file src/utils/dataProcessor.optimized-backup-20260101_120000.ts
   ```

3. **Cache corruption recovery**:
   ```bash
   rm ~/.code-augur/pattern-cache.db
   code-augur analyze --rebuild-cache
   ```

4. **Full system reset** (if predictions become unusable):
   ```bash
   rm -rf ~/.code-augur/
   # Clear npm cache if AST parser issues:
   npm cache clean --force
   ```

5. **Model version rollback** (if new OpenAI model breaks predictions):
   ```bash
   export CODE_AUGUR_MODEL="gpt-3.5-turbo"
   # Add to .env to persist
   echo "CODE_AUGUR_MODEL=gpt-3.5-turbo" >> .env
   ```

6. **Selective pattern disabling**:
   If a specific pattern causes false positives:
   ```bash
   # Add to ~/.code-augur/config.yaml:
   disabled_patterns:
     - "async-await-without-try-catch:src/legacy/"
     - "manual-debounce:src/animations/"
   ```

7. **Rollback entire session** (if multiple predictions applied):
   ```bash
   # If you created a branch for predictions:
   git checkout main
   git branch -D feature/augur-predictions
   # Or revert merge if already merged:
   git revert -m 1 <merge-commit-hash>
   ```

8. **Emergency stop** (if prediction engine hangs):
   ```bash
   pkill -f "code-augur"
   # Clear temporary files:
   rm -rf /tmp/code-augur-*/
   ```

Verification after rollback:
```bash
# Ensure no leftover generated files
git status | grep "generated" && echo "Cleanup needed"
# Rebuild cache with known-good codebase version
code-augur analyze --validate-only
```
```