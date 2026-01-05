---
id: project-setup-linters-ci-ai
title: Production-Grade Project Setup with Linters, Spotless, Detekt, and CI for AI Collaboration
tldr: Properly configured linters (Spotless, Detekt) with automated CI workflows create consistent, maintainable code that works seamlessly with AI tools like Claude Code, enabling safe, high-velocity development.
tags: [ci-cd, linting, code-quality, detekt, spotless, github-actions, automation]
difficulty: advanced
readTimeMinutes: 14
publishedDate: null
relatedTopics: [supervised-scope-build-plugins, robolectric-unit-testing]
---

## Why Linters Matter for AI-Assisted Development

AI tools like Claude Code generate code quickly, but without proper guardrails, you get:
- Inconsistent formatting
- Potential security issues
- Style violations
- Import chaos

**The solution:** Automated linting and formatting that:
- Runs before commit (pre-commit hooks)
- Executes in CI (blocks merge on failure)
- Auto-fixes where possible
- Provides consistent codebase for AI to learn from

## The Trifecta: Spotless + Detekt + ktlint

### Spotless: Code Formatting

Spotless enforces consistent formatting automatically.

```kotlin
// build-logic/convention/src/main/kotlin/spotless-convention.gradle.kts
import com.diffplug.gradle.spotless.SpotlessExtension

plugins {
    id("com.diffplug.spotless")
}

configure<SpotlessExtension> {
    kotlin {
        target("src/**/*.kt")
        targetExclude("build/**/*.kt")

        ktlint("1.1.1").apply {
            setEditorConfigPath("$rootDir/.editorconfig")
        }

        // Custom formatting rules
        trimTrailingWhitespace()
        endWithNewline()

        // License header
        licenseHeaderFile(rootProject.file("spotless/copyright.kt"))
    }

    kotlinGradle {
        target("*.gradle.kts")
        ktlint("1.1.1")
    }

    format("xml") {
        target("src/**/*.xml")
        trimTrailingWhitespace()
        endWithNewline()
        eclipseWtp(EclipseWtpFormatterStep.XML)
    }
}
```

### Detekt: Static Code Analysis

Detekt catches code smells, potential bugs, and complexity issues.

```kotlin
// build-logic/convention/src/main/kotlin/detekt-convention.gradle.kts
import io.gitlab.arturbosch.detekt.Detekt

plugins {
    id("io.gitlab.arturbosch.detekt")
}

detekt {
    buildUponDefaultConfig = true
    allRules = false // Enable all rules
    config.setFrom(files("$rootDir/config/detekt/detekt.yml"))
    baseline = file("$rootDir/config/detekt/baseline.xml")

    reports {
        html.required.set(true)
        xml.required.set(true)
        txt.required.set(true)
        sarif.required.set(true)
        md.required.set(true)
    }
}

tasks.withType<Detekt>().configureEach {
    jvmTarget = "17"

    reports {
        html.required.set(true)
        xml.required.set(true)
        txt.required.set(true)
    }

    // Parallel execution for speed
    setMaxWorkers(Runtime.getRuntime().availableProcessors())
}

// Type resolution for better analysis
dependencies {
    detektPlugins("io.gitlab.arturbosch.detekt:detekt-formatting:1.23.4")
    detektPlugins("io.gitlab.arturbosch.detekt:detekt-rules-libraries:1.23.4")
}
```

### Detekt Configuration

```yaml
# config/detekt/detekt.yml
build:
  maxIssues: 0 # Fail build if any issues
  excludeCorrectable: false # Include auto-fixable issues

processors:
  active: true

complexity:
  active: true
  CyclomaticComplexMethod:
    active: true
    threshold: 15
  LongMethod:
    active: true
    threshold: 60
  LongParameterList:
    active: true
    functionThreshold: 6
    constructorThreshold: 7
  TooManyFunctions:
    active: true
    thresholdInFiles: 25
    thresholdInClasses: 25

coroutines:
  active: true
  GlobalCoroutineUsage:
    active: true
  SuspendFunWithFlowReturnType:
    active: true

exceptions:
  active: true
  TooGenericExceptionCaught:
    active: true
  SwallowedException:
    active: true
  ExceptionRaisedInUnexpectedLocation:
    active: true

naming:
  active: true
  FunctionNaming:
    active: true
    functionPattern: '[a-z][a-zA-Z0-9]*'
  ClassNaming:
    active: true
    classPattern: '[A-Z][a-zA-Z0-9]*'
  VariableNaming:
    active: true
    variablePattern: '[a-z][a-zA-Z0-9]*'

performance:
  active: true
  SpreadOperator:
    active: true
  ForEachOnRange:
    active: true

potential-bugs:
  active: true
  UnsafeCast:
    active: true
  EqualsWithHashCodeExist:
    active: true
  DuplicateCaseInWhenExpression:
    active: true

style:
  active: true
  UnusedImports:
    active: true
  MagicNumber:
    active: true
    ignoreAnnotation: true
    ignoreHashCodeFunction: true
  MaxLineLength:
    active: true
    maxLineLength: 120
  ReturnCount:
    active: true
    max: 3
```

## Pre-Commit Hooks with Husky/Git Hooks

Catch issues before they reach CI.

```bash
# .git/hooks/pre-commit
#!/bin/sh

echo "Running pre-commit checks..."

# Run spotless check
./gradlew spotlessCheck
if [ $? -ne 0 ]; then
  echo "❌ Spotless check failed. Run './gradlew spotlessApply' to fix formatting."
  exit 1
fi

# Run detekt
./gradlew detekt
if [ $? -ne 0 ]; then
  echo "❌ Detekt found issues. Fix them before committing."
  exit 1
fi

echo "✅ Pre-commit checks passed"
exit 0
```

Make it executable:

```bash
chmod +x .git/hooks/pre-commit
```

## Gradle Tasks for Automation

Create composite tasks for common workflows:

```kotlin
// build.gradle.kts
tasks.register("check-code-quality") {
    group = "verification"
    description = "Run all code quality checks"

    dependsOn(
        "spotlessCheck",
        "detekt",
        "lintDebug" // Android lint if applicable
    )
}

tasks.register("fix-code-quality") {
    group = "verification"
    description = "Auto-fix code quality issues"

    dependsOn(
        "spotlessApply"
    )

    doLast {
        println("✅ Auto-fixable issues resolved. Run 'detekt' to check remaining issues.")
    }
}
```

## CI/CD Configuration

### GitHub Actions Workflow

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on:
  pull_request:
    branches: [ main, develop ]

jobs:
  code-quality:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Full history for detekt baseline

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Check formatting (Spotless)
        run: ./gradlew spotlessCheck

      - name: Run static analysis (Detekt)
        run: ./gradlew detekt

      - name: Upload Detekt reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: detekt-reports
          path: build/reports/detekt/

      - name: Comment PR with issues
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('build/reports/detekt/detekt.md', 'utf8');

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## ❌ Code Quality Issues\n\n${report}`
            });

  tests:
    runs-on: ubuntu-latest
    needs: code-quality # Run after quality checks pass

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle

      - name: Run tests
        run: ./gradlew test

      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: '**/build/test-results/**/*.xml'

      - name: Upload test coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: '**/build/reports/jacoco/**/*.xml'
```

### Parallel Job Execution

```yaml
# .github/workflows/fast-checks.yml
name: Fast Quality Checks

on: [push, pull_request]

jobs:
  spotless:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle
      - run: ./gradlew spotlessCheck

  detekt:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle
      - run: ./gradlew detekt

  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: gradle
      - run: ./gradlew test
```

## EditorConfig for Consistency

```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.{kt,kts}]
indent_style = space
indent_size = 4
max_line_length = 120
ij_kotlin_allow_trailing_comma = true
ij_kotlin_allow_trailing_comma_on_call_site = true

[*.{xml,json,yml,yaml}]
indent_style = space
indent_size = 2

[*.md]
trim_trailing_whitespace = false
```

## Danger for PR Automation

Automate PR reviews with Danger:

```kotlin
// Dangerfile.kts
import systems.danger.kotlin.*

danger(args) {
    val allSourceFiles = git.modifiedFiles + git.createdFiles

    // Warn about large PRs
    if (allSourceFiles.size > 50) {
        warn("This PR is quite large (${allSourceFiles.size} files). Consider splitting it.")
    }

    // Require tests for new code
    val hasAppChanges = allSourceFiles.any { it.contains("src/main/") }
    val hasTestChanges = allSourceFiles.any { it.contains("src/test/") }

    if (hasAppChanges && !hasTestChanges) {
        warn("You've added code but no tests. Consider adding tests.")
    }

    // Enforce changelog updates
    val hasChangelog = git.modifiedFiles.contains("CHANGELOG.md")
    if (!hasChangelog && hasAppChanges) {
        warn("Please update CHANGELOG.md with your changes.")
    }

    // Check for TODO comments in new code
    val newCodeWithTodo = git.createdFiles.filter { file ->
        val content = File(file).readText()
        content.contains("TODO", ignoreCase = true)
    }

    if (newCodeWithTodo.isNotEmpty()) {
        warn("New files contain TODO comments: ${newCodeWithTodo.joinToString()}")
    }
}
```

## Custom Detekt Rules

Create project-specific rules:

```kotlin
// detekt-custom-rules/src/main/kotlin/NoGlobalCoroutineScope.kt
class NoGlobalCoroutineScope(config: Config) : Rule(config) {
    override val issue = Issue(
        javaClass.simpleName,
        Severity.Defect,
        "Using GlobalScope is discouraged. Use structured concurrency instead.",
        Debt.TEN_MINS
    )

    override fun visitCallExpression(expression: KtCallExpression) {
        super.visitCallExpression(expression)

        if (expression.text.contains("GlobalScope")) {
            report(
                CodeSmell(
                    issue,
                    Entity.from(expression),
                    "GlobalScope usage detected. Use viewModelScope or lifecycleScope instead."
                )
            )
        }
    }
}
```

Register custom rule provider:

```kotlin
// detekt-custom-rules/src/main/kotlin/CustomRuleSetProvider.kt
class CustomRuleSetProvider : RuleSetProvider {
    override val ruleSetId: String = "custom-rules"

    override fun instance(config: Config): RuleSet {
        return RuleSet(
            ruleSetId,
            listOf(
                NoGlobalCoroutineScope(config),
                RequireViewModelFactory(config),
                NoHardcodedStrings(config)
            )
        )
    }
}
```

## AI-Friendly Project Structure

Help AI tools understand your project:

```markdown
# CLAUDE.md
# This file provides guidance to Claude Code when working with this repository.

## Project Overview
This is a Kotlin Multiplatform project targeting Android, iOS, and Desktop.

## Build Commands
\`\`\`bash
# Run tests
./gradlew test

# Format code
./gradlew spotlessApply

# Run linting
./gradlew detekt

# Full check
./gradlew check-code-quality
\`\`\`

## Code Style
- Max line length: 120
- Use trailing commas
- Follow detekt rules in config/detekt/detekt.yml
- All public APIs must have KDoc comments

## Testing
- Unit tests in src/test/
- Use Robolectric for Android-specific tests
- Prefer fakes over mocks
- Aim for 80%+ code coverage

## CI/CD
- All PRs must pass spotless + detekt + tests
- Pre-commit hooks enforce quality checks locally
- GitHub Actions run on every PR

## Architecture
- Follow Clean Architecture
- Repository pattern for data access
- MVVM/MVI for presentation
- Use Kotlin Flow for reactive streams
```

## Benefits for AI Collaboration

Properly configured linters and CI create an environment where AI tools excel:

**1. Consistent Context**: AI learns from consistent code patterns
**2. Fast Feedback**: Pre-commit hooks catch issues instantly
**3. Auto-Fix**: Spotless automatically fixes formatting
**4. Clear Rules**: Detekt provides explicit guidelines
**5. Safe Merges**: CI ensures code quality before merge

## Conclusion

A well-configured project with Spotless, Detekt, pre-commit hooks, and CI/CD:
- **Catches issues early** before they reach production
- **Maintains consistency** across the team and AI contributions
- **Speeds up reviews** with automated checks
- **Enables safe collaboration** with AI tools
- **Reduces technical debt** through continuous quality enforcement

This infrastructure transforms your project into a well-oiled machine that welcomes both human and AI contributions while maintaining high code quality standards.
