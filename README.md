# Intelligent Test Generator

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> Auto-generates unit, integration, and e2e tests from source code and specs — with AI-driven coverage gap analysis, mutation-score feedback, and automatic test repair.

An open-source, AI-native test generation platform targeting polyglot codebases (Python, TypeScript, Java, Go). It is built for backend engineers and QA leads who need to close coverage gaps systematically, for DevOps teams enforcing coverage thresholds in CI/CD pipelines, and for open-source maintainers who cannot afford dedicated QA resources. Where incumbent tools either lock into a single language or treat test generation as a one-shot autocomplete suggestion, this project treats test quality as an iterative, measurable outcome.

---

## Why Intelligent Test Generator?

- **No production-grade open-source polyglot option exists.** Diffblue Cover is Java-only and commercial. Keploy requires a live running application. EvoSuite is research-quality and Java-only. No tool covers Python, TypeScript, Go, and Java in a single production-ready open-source package.
- **Incumbent pricing excludes small teams.** Qodo Teams costs $30/user/month; Diffblue scales with lines of code and reaches enterprise pricing quickly. The $0 open-source tier (EvoSuite, Keploy, Qodo Cover) covers only narrow use cases.
- **Coverage metrics are gamed, not quality metrics.** Every existing generator maximises line or branch coverage — a criterion that can be satisfied with trivial assertions that pass but verify nothing. Mutation score is the correct quality signal; no open-source generator targets it by default.
- **Tests break constantly; no OSS tool repairs them.** Diffblue Cover auto-maintains tests when code changes — a unique capability in the commercial space with no open-source equivalent. Developer time lost to test maintenance is a well-documented productivity sink.
- **Spec-to-test generation is unsolved.** All current tools generate tests from existing code only. Teams practising TDD or working from OpenAPI specs have no tool that generates tests before the implementation is written.

---

## Key Features

### Coverage Gap Analysis

- Identifies which functions, branches, and logical paths lack test coverage before generating anything
- Explains *why* a path is hard to reach (complex setup, external dependencies, concurrency) and generates the scaffolding needed to make it reachable
- Iterative prompting mode focuses token usage only on uncovered branches, reducing cost and improving yield

### Multi-Language Unit Test Generation

- Generates tests for Python, TypeScript/JavaScript, Java, and Go from source code
- Output targets standard runners: pytest, Jest, JUnit 4/5, and Go Test
- Context-aware mock and fixture generation for external dependencies — no manual mock setup

### Mutation-Score Feedback Loop

- Runs mutation testing after initial generation, identifies surviving mutants, and generates additional assertions specifically targeting those mutants
- Produces test suites with genuine defect-detection power, not just coverage percentage
- Mutation score dashboard per module with historical trend tracking

### Spec-to-Test Generation

- Generates tests from OpenAPI 3.x specs, Gherkin feature files, and natural-language requirements before implementation exists
- Catches requirement ambiguities at the specification stage rather than post-implementation
- Enables true TDD at scale without per-developer discipline

### IDE and CI/CD Integration

- VS Code and JetBrains IDE plugins for inline test generation without leaving the editor
- CLI with coverage threshold gates for merge blocking as a CI pipeline stage
- Coverage trend reporting across commits with regression alerts

### Test Maintenance and Auto-Repair

- Detects broken tests after production code changes and auto-repairs them using semantic diff analysis
- Distinguishes intentional behavioural changes from unintended regressions
- Flags cases where a failing test reveals a real bug rather than a stale assertion

---

## AI-Native Advantage

Rule-based generators (EvoSuite, EvoMaster) maximise structural coverage criteria but cannot reason about *why* code is hard to test or what a test should actually verify. LLM-based generation alone (GitHub Copilot inline suggestions) treats tests as autocomplete rather than a systematic quality outcome. This project combines LLM-driven semantic understanding with a generation-validation-repair loop — the architecture validated by ChatUniTest (59.6% line coverage, outperforming EvoSuite on 50% of evaluated projects) — and adds a mutation-testing feedback signal to iteratively improve assertion strength. The result is a system that improves test quality autonomously without human intervention, an approach demonstrated at production scale by Meta's internal LLM+mutation testing deployment reported at FSE 2025.

---

## Tech Stack & Deployment

The tool is designed for self-hosted deployment as a CLI, VS Code/JetBrains plugin, and optional CI service. The core generation pipeline is LLM-agnostic, targeting the Anthropic Claude API by default with an interface layer permitting substitution. Test output targets existing runners (pytest, Jest, JUnit, Go Test) with no lock-in to a proprietary execution environment. Mutation testing integration leverages established tools (PITest for Java, mutmut for Python, Stryker for TypeScript) via subprocess rather than reimplementing mutation operators. CI integration is delivered as a GitHub Actions action and a generic CLI that any pipeline can shell out to.

---

## Market Context

The AI-enabled testing market was valued at ~$1.01B in 2025, growing at an 18.3% CAGR; the broader automation testing market reached $24–40B in 2026 (Mordor Intelligence, Fortune Business Insights). Primary buyers are backend engineers and QA leads at product companies with low test coverage (<40%), DevOps/platform teams enforcing coverage gates, and security-focused engineering organisations requiring OWASP OTG-mapped test cases. Commercial alternatives range from $19–$30/user/month (Qodo, GitHub Copilot) to enterprise-priced LoC-based contracts (Diffblue), creating a clear opening for a production-grade open-source alternative at $0.

---

## Project Status

> This project is in the **research and specification phase**.
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

All contributions must be your own original work or clearly attributed open-source material with a compatible licence. Copyright infringement and licence violations will not be tolerated and will result in immediate removal of the offending contribution. If you are unsure whether a piece of code, text, or other material is safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined.
