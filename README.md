# 🤖 Copilot Dev Kit

> **A comprehensive multi-agent development system for GitHub Copilot that coordinates specialized AI agents to plan, implement, and document software features with enterprise-grade quality.**

[![GitHub Copilot](https://img.shields.io/badge/GitHub-Copilot-blue?logo=github)](https://github.com/features/copilot)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

---

## 📋 Overview

**Copilot Dev Kit** is a structured, agent-based workflow system that extends GitHub Copilot's capabilities through specialized AI agents. Each agent has a specific role in the software development lifecycle, from strategic planning to code implementation and documentation.

### Why This Matters

- **Consistency**: Every feature follows the same high-quality process
- **Traceability**: Complete history of decisions and implementations
- **Specialization**: Right agent for the right task
- **Best Practices**: Built-in coding standards, architecture patterns, and performance guidelines
- **Memory**: Never lose context between sessions

---

## 🎯 Core Agents

The system is built around four specialized agents, orchestrated by Skynet:

### 🎛️ **Skynet** — The Orchestrator
- **Role**: Coordinates all agents and manages workflow
- **Never**: Writes code or creates plans directly
- **Responsibilities**:
  - Load project history at session start
  - Delegate to specialized agents
  - Ensure workflow consistency
  - Track feature completion

### 🖖 **Spock** — Strategic Planner & Architect
- **Role**: Creates implementation strategies and technical plans
- **Process**: Two-phase workflow (Interview → Plan)
- **Responsibilities**:
  - Gather requirements through structured interviews
  - Analyze codebase and identify patterns
  - Produce detailed implementation plans
  - Coordinate UI/UX design (via Woz) when needed
  - Assess technical risks and dependencies

### 💻 **Neo** — Expert Coder
- **Role**: Writes production-ready code
- **Principles**: Clean Code, SOLID, DDD, performance-first
- **Responsibilities**:
  - Implement features following Spock's plans
  - Write directly to files (never code in chat)
  - Follow language/framework-specific best practices
  - Create tests alongside implementation
- **Supported Stacks**:
  - C# / .NET (Minimal API, EF Core, xUnit)
  - TypeScript / Angular (Signals, Standalone Components)
  - Python, React, and more

### 📚 **Alexandria** — Project Memory
- **Role**: Living documentation and project history
- **Responsibilities**:
  - Load context at session start
  - Track implemented, partial, and planned features
  - Update plan files and changelog on completion
  - Maintain architectural decision records

### 🎨 **Woz** — UI/UX Designer
- **Role**: Creates design specifications and UI/UX guidelines
- **Called by**: Spock during planning phase
- **Responsibilities**:
  - Design user interfaces and interactions
  - Define component hierarchies
  - Specify accessibility requirements

---

## 🚀 Quick Start

### 1. Setup

Clone this repository into your project's `.github` directory:

```bash
# In your project root
git clone https://github.com/mgattilabs/copilot_dev_kit .github
```

Or add as a submodule:

```bash
git submodule add https://github.com/mgattilabs/copilot_dev_kit .github/copilot_dev_kit
```

### 2. Invoke Skynet

In GitHub Copilot Chat (VS Code), simply type:

```
@skynet implement user authentication with JWT
```

Skynet will:
1. Load project history via Alexandria
2. Delegate to Spock for planning
3. Once plan is approved, delegate to Neo for implementation
4. Update documentation via Alexandria on completion

### 3. Review Plan Files

All plans are stored in `docs/plan/` with ISO timestamps:

```
docs/plan/2026-02-24-1430-jwt-authentication.md
```

---

## 📂 Project Structure

```
.github/
├── agents/                      # Agent definitions
│   ├── skynet.agent.md         # Orchestrator
│   ├── spock.agent.md          # Planner
│   ├── neo.agent.md            # Coder
│   ├── alexandria.agent.md     # Memory
│   └── woz.agent.md            # Designer
│
├── instructions/               # Generic coding guidelines
│   ├── code-review-generic.instructions.md
│   ├── performance-optimization.instructions.md
│   ├── context-engineering.instructions.md
│   ├── csharp.instructions.md
│   ├── angular.instructions.md
│   ├── dotnet-architecture-good-practices.instructions.md
│   ├── object-calisthenics.instructions.md
│   └── azure-devops-pipelines.instructions.md
│
└── skills/                     # Domain-specific expertise
    ├── neo-dotnet/            # .NET implementation patterns
    ├── neo-angular/           # Angular best practices
    ├── spock-architecture/    # Architecture selection & CQRS
    ├── spock-domain-analysis/ # Domain modeling
    └── spock-risk-assessment/ # Migration & risk analysis

docs/                           # Created in your project
├── plan/                       # Implementation plans
│   └── YYYY-MM-DD-HHmm-feature-name.md
└── IMPLEMENTATION-LOG.md       # Changelog of all features
```

---

## 🔄 Workflow Example

### Scenario: Implement Minimal API Endpoints

```
User: "@skynet convert controllers to minimal API"

1. Skynet → Alexandria (open mode)
   Returns: Project history, existing patterns, dependencies

2. Skynet → Spock (interview mode)
   Spock asks: Which controllers? Performance requirements? Breaking changes?

3. User answers questions

4. Skynet → Spock (plan mode)
   Spock produces:
   - docs/plan/2026-02-24-1430-minimal-api-conversion.md
   - Technical strategy
   - File-by-file breakdown
   - Testing strategy
   - Risk assessment

5. User reviews plan → approves

6. Skynet → Neo (implement mode, with plan)
   Neo writes:
   - Minimal API endpoints
   - DTOs and validators
   - Unit tests
   - Integration tests

7. Skynet → Alexandria (close mode)
   Alexandria updates:
   - Plan file status → "✅ Implemented"
   - docs/IMPLEMENTATION-LOG.md with entry
```

**Result**: Feature implemented, tested, and documented in < 10 minutes.

---

## 🛠️ Customization

### Add Your Own Instructions

Place custom instructions in `.github/instructions/`:

```markdown
---
description: 'Project-specific coding rules'
applyTo: '**/*.cs'
---

# My Project Rules

- Always use record types for DTOs
- Prefix interfaces with 'I'
- Use minimal API endpoints pattern
```

### Add Domain Skills

Create skills in `.github/skills/[agent-name]-[domain]/`:

```
.github/skills/neo-myframework/
├── SKILL.md                    # Skill definition
└── references/
    ├── patterns.md
    └── examples.md
```

### Configure Agents

Edit agent files in `.github/agents/` to:
- Add/remove tools
- Adjust descriptions
- Change models
- Add new agents

---

## 📊 Documentation & Memory

### Plan Files

Every feature gets a plan file in `docs/plan/`:

```markdown
---
Progetto: MyApp
Data: 2026-02-24
Tipo: Feature Implementation
Obiettivo: Convert Controllers to Minimal API
---

## Descrizione
[Strategic overview]

## Analisi Tecnica
[Technical decisions]

## Piano di Implementazione
[Step-by-step breakdown]

## Testing Strategy
[Test approach]

## Status
✅ Implemented
**Completato**: 2026-02-24 14:30 UTC
```

### Implementation Log

`docs/IMPLEMENTATION-LOG.md` tracks all features:

| Data | Feature | Plan | Status | Note |
|------|---------|------|--------|------|
| 2026-02-24 | Minimal API Conversion | [link](plan/...) | ✅ | Completed |

---

## 🎓 Best Practices Built-In

### Code Quality
- **SOLID Principles**: Enforced through instructions
- **Clean Code**: Naming, structure, readability
- **Object Calisthenics**: 9 rules for better OOP
- **DRY**: No code duplication

### Architecture
- **CQRS**: Command/Query separation patterns
- **Clean Architecture**: Dependency inversion
- **DDD**: Domain-driven design patterns
- **Modular Monolith**: Scalable without microservices

### Performance
- **Database**: Query optimization, indexing, N+1 prevention
- **Frontend**: Lazy loading, tree-shaking, bundle optimization
- **Backend**: Caching, async I/O, connection pooling
- **Profiling**: Built-in performance benchmarking guidance

### Testing
- **Unit Tests**: Arrange-Act-Assert pattern
- **Integration Tests**: Real dependencies
- **Coverage**: Critical paths must be tested
- **Test Naming**: Descriptive, scenario-based names

---

## 🤝 Contributing

Contributions are welcome! Areas of interest:

- New agent types (e.g., Security Auditor, Performance Analyst)
- Additional domain skills (React, Vue, Go, Rust, etc.)
- Language-specific instructions (Java, Ruby, Swift, etc.)
- Enhanced workflow patterns
- Bug fixes and improvements

**How to contribute:**
1. Fork this repository
2. Create a feature branch
3. Make your changes following existing patterns
4. Submit a pull request with clear description

---

## 📖 Documentation

- [Agent Architecture](docs/AGENT-ARCHITECTURE.md) — How agents interact
- [Workflow Guide](docs/WORKFLOW-GUIDE.md) — Detailed execution patterns
- [Custom Skills Guide](docs/CUSTOM-SKILLS.md) — Creating domain expertise
- [Troubleshooting](docs/TROUBLESHOOTING.md) — Common issues and solutions

---

## 🔗 Resources

- [GitHub Copilot Documentation](https://docs.github.com/en/copilot)
- [Copilot Custom Instructions](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)
- [Prompt Engineering Guide](https://docs.github.com/en/copilot/concepts/prompting/prompt-engineering)
- [Awesome GitHub Copilot](https://github.com/github/awesome-copilot)

---

## 📄 License

MIT License - see [LICENSE](LICENSE) file for details

---

## 🙏 Acknowledgments

Built with inspiration from:
- Clean Code by Robert C. Martin
- Domain-Driven Design by Eric Evans
- Microservices Patterns by Chris Richardson
- The Pragmatic Programmer by Hunt & Thomas

---

## 📧 Contact

**Author**: Marco Gatti  
**Organization**: MGatti Labs  
**Repository**: [mgattilabs/copilot_dev_kit](https://github.com/mgattilabs/copilot_dev_kit)

For questions, issues, or suggestions, please [open an issue](https://github.com/mgattilabs/copilot_dev_kit/issues) on GitHub.

---

<div align="center">

**Made with ❤️ for developers who care about code quality**

⭐ Star this repo if you find it useful!

</div>