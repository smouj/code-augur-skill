name: code-augur
version: 2.1.0
description: Motor de predicción y optimización de patrones de código impulsado por IA para proyectos TypeScript/JavaScript/Go
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

Motor inteligente de predicción y optimización de patrones de código que analiza codebases existentes para predecir implementaciones futuras, sugerir mejoras arquitectónicas y generar soluciones optimizadas basadas en patrones establecidos del proyecto.

## Purpose

Casos de uso reales:
- **API Endpoint Prediction**: Cuando comienzas un nuevo endpoint REST, Code Augur predice la pila completa controller/service/repository basándose en patrones existentes en el proyecto.
- **Testing Pattern Generation**: Genera automáticamente la estructura de archivos de prueba y boilerplate que coinciden con el framework de testing existente del proyecto (Jest, Vitest, Go test, etc.).
- **Migration Pattern Detection**: Identifica cuando el código está implementando manualmente patrones que tienen equivalentes en librerías (por ejemplo, debouncing manual cuando lodash.debounce ya es una dependencia).
- **Performance Anti-Pattern Discovery**: Escanea para queries N+1, re-renders innecesarios en React, loops ineficientes, y sugiere alternativas optimizadas usadas en otras partes del codebase.
- **Consistency Enforcement**: Detecta cuando componentes/funciones similares usan patrones diferentes y sugiere estandarizar al enfoque dominante.
- **Boilerplate Generation**: Crea nuevos módulos siguiendo exactamente las convenciones del proyecto: manejo de errores, logging, validación, definiciones de tipos y exports.

## Alcance

Comandos que pueden ser ejecutados:
- `code-augur analyze [path] [--extensions] [--exclude] [--depth]`
- `code-augur predict [--file] [--line] [--context] [--output]`
- `code-augur optimize [--target] [--strategy] [--dry-run]`
- `code-augur suggest [--issue-type] [--module] [--format]`
- `code-augur validate [--pattern] [--test] [--coverage]`
- `code-augur learn [--examples] [--categories] [--weight]`
- `code-augur export [--format json|yaml|md] [--include-stats]`

## Proceso de Trabajo

Pasos detallados de ejecución:

1. **Environment Validation**
   ```bash
   # Verify dependencies and API access
   node -e \"console.log(require('ast-parser').version)\"
   python3 -c \"import pattern_matcher; print(pattern_matcher.__version__)\"
   test -n \"$OPENAI_API_KEY\" || { echo \"Missing OPENAI_API_KEY\"; exit 1; }
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
     --context \"POST endpoint for user registration with validation\" \
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
     --pattern \"async-await-without-try-catch\" \
     --test \"detects patterns in isolation mode\" \
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

## Reglas de Oro

1. **Nunca modifiques código en producción sin --dry-run primero**. Siempre revisa los cambios generados y ejecuta tests.
2. **Invalidación de caché**: Cuando las dependencias cambien, ejecuta `code-augur analyze --rebuild-cache` para evitar predicciones obsoletas.
3. **Umbral de confianza**: Ignora predicciones con confianza < 0.75 a menos que se sobrescriba explícitamente con `--force`.
4. **Detección de drift de patrones**: Semanalmente, ejecuta `code-augur validate --drift-check` para asegurar que las predicciones aún coinciden con el codebase en evolución.
5. **Seguridad de secretos**: Nunca envíes código que contenga API keys, passwords o tokens al motor de predicción. Usa `--sanitize-secrets` (habilitado por defecto) para reemplazar con placeholders.
6. **Presupuesto de rendimiento**: Si el análisis toma >30 segundos, reduce el alcance con `--depth` o `--extensions`.
7. **Revisión humana obligatoria** para cualquier optimización que toque manejo de errores, verificaciones de seguridad o validación de datos.
8. **Version lock**: La caché está vinculada al hash del `package.json` y `go.mod` del proyecto. Si las dependencias cambian significativamente, limpia la caché en `~/.code-augur/pattern-cache.db`.

## Examples

### Example 1: Predict a new React component pattern
```bash
# Existing: Button, Input, Modal components use a factory pattern with forwardRef
code-augur predict \
  --file src/components/Select.tsx \
  --line 8 \
  --context \"Create a select dropdown component with options prop and onChange handler\" \
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
        className=\"select-base\"
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
code-augur suggest --issue-type \"deprecated-pattern\" --module src/auth/
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

Procedimientos específicos de rollback:

1. **Deshacer generación de predicción** (antes de commit):
   ```bash
   # Si ejecutaste predict con --output pero no has hecho commit
   rm generated-snippet.ts
   # O si usaste el flag --in-place:
   git checkout -- src/controllers/UserController.ts
   ```

2. **Restaurar desde backup de optimización**:
   Code Augur crea automáticamente `file.optimized-backup-<timestamp>.ts` cuando se usa `--in-place`.
   ```bash
   code-augur restore --backup-file src/utils/dataProcessor.optimized-backup-20260101_120000.ts
   ```

3. **Recuperación de corrupción de caché**:
   ```bash
   rm ~/.code-augur/pattern-cache.db
   code-augur analyze --rebuild-cache
   ```

4. **Reset completo del sistema** (si las predicciones se vuelven inutilizables):
   ```bash
   rm -rf ~/.code-augur/
   # Limpia npm cache si hay problemas con AST parser:
   npm cache clean --force
   ```

5. **Rollback de versión del modelo** (si un nuevo modelo de OpenAI rompe las predicciones):
   ```bash
   export CODE_AUGUR_MODEL=\"gpt-3.5-turbo\"
   # Añade a .env para persistir
   echo \"CODE_AUGUR_MODEL=gpt-3.5-turbo\" >> .env
   ```

6. **Deshabilitación selectiva de patrones**:
   Si un patrón específico causa falsos positivos:
   ```bash
   # Añade a ~/.code-augur/config.yaml:
   disabled_patterns:
     - \"async-await-without-try-catch:src/legacy/\"
     - \"manual-debounce:src/animations/\"
   ```

7. **Rollback de sesión completa** (si se aplicaron múltiples predicciones):
   ```bash
   # Si creaste una rama para predicciones:
   git checkout main
   git branch -D feature/augur-predictions
   # O revertir merge si ya fue mergeado:
   git revert -m 1 <merge-commit-hash>
   ```

8. **Detención de emergencia** (si el motor de predicción se cuelga):
   ```bash
   pkill -f \"code-augur\"
   # Limpia archivos temporales:
   rm -rf /tmp/code-augur-*/
   ```

Verificación después del rollback:
```bash
# Asegúrate que no queden archivos generados
git status | grep \"generated\" && echo \"Cleanup needed\"
# Reconstruye caché con la versión conocida-good del codebase
code-augur analyze --validate-only
```
```