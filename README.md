# Smart Task for Azure DevOps Pipelines

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0+-blue.svg)](https://www.typescriptlang.org/)

Overview
--------

Smart Task is an **AI-based decision support engine** for Azure DevOps pipelines. It helps pipelines make **context-aware decisions at runtime** - such as determining test execution strategy, choosing security scanning depth, or assessing deployment risk—based on what actually changed and the surrounding pipeline context.

Smart Task **does not execute pipeline steps**. It resolves decisions and exposes them as **decision signals** (also referred to as decision pointers) in the form of pipeline variables that standard Azure DevOps pipeline logic can consume deterministically.

### Why Smart Task?

Modern CI/CD pipelines are excellent at execution, but they struggle with decisions that depend on runtime context, such as:

- Which tests should run for this change?
- Should expensive or compliance-driven security scans be executed?
- Does this deployment candidate represent low, medium, or high risk?
- Should stricter controls be applied for this pipeline run?

These decisions are commonly implemented using:
- Static rules
- Copied templates
- Manual runtime flags/parameters

Over time, this leads to **inconsistent behavior, higher costs, and operational friction** across teams.

Smart Task centralizes this logic into a reusable **decision support layer**, while keeping execution and governance fully in the pipeline.


### Key Characteristics

- **Decision support only**: Smart Task never executes commands or performs pipeline actions.

- **Explicit decision signals**: Decisions are emitted as named pipeline variables.

- **Deterministic by design**: Pipelines remain predictable when required decision signals are explicitly defined.

- **Transparent outputs**: Decisions include confidence and reasoning to support review and trust.

- **Human-in-the-loop friendly**: Pipelines decide how to act on decisions using native Azure DevOps mechanisms.

- **Extensible context model**: Context can be enriched via built-in tools and additional information provided from external sources.

Architecture
------------

At a high level:

1.  The Azure DevOps task invokes the **Decision Support Engine**
    
2.  The engine invokes a **Decision Workflow**
    
3.  The workflow runs a **Plan → Execute → Replan** loop (orchestrated using LangGraph)
    
4.  A final decision result is produced
    
5.  The engine relays the result as **decision signals (pipeline variables)**
    

The pipeline remains fully responsible for:

*   Execution
    
*   Approvals
    
*   Overrides
    
*   Governance
    

Decision Workflow (Plan → Execute → Replan)
-------------------------------------------

Smart Task uses a structured decision workflow orchestrated with **LangGraph**:

1.  **Plan**Interprets the prompt and identifies required decision signals and inputs.
    
2.  **Execute**Gathers context using tools (pipeline metadata, changed files, environment details, custom sources).
    
3.  **Replan**Uses an LLM to reason over the gathered signals, correct assumptions if needed, and converge on stable decisions.
    

The final output includes:

*   Decision signals
    
*   Confidence levels
    
*   Reasoning

Important: Deterministic Decision Signals
-----------------------------------------

To ensure **deterministic and predictable behavior**, the **required decision signals must be explicitly specified in the prompt**.

Smart Task does **not infer** which signals the pipeline depends on.

If the prompt is open-ended, the output signals may vary across runs.

### Recommended pattern for prompt

```text
Determine the following decision signals:
- SIGNAL_1
- SIGNAL_2
- CONFIDENCE
- REASONING
```

This keeps responsibility clear:

*   The **prompt defines required signals**
    
*   The **decision engine resolves them**
    
*   The **pipeline consumes them**

Human-in-the-Loop Design
------------------------

Smart Task **does not decide governance**.

Instead:

*   It emits decision signals with confidence and reasoning
    
*   Pipelines map these signals to:
    
    *   Approvals
        
    *   Manual validation
        
    *   Overrides
        

This ensures:

*   Deterministic execution
    
*   Explicit governance
    
*   Humans remain in control

Built-In Tools
---------------

Smart Task includes a set of **specialized tools** used by the decision workflow to gather relevant signals from the pipeline, repository, and execution environment. These tools are **context-focused and observational** in nature, and are used exclusively to support decision-making within the Plan → Execute → Replan workflow.

Smart Task does **not** use these tools to control pipeline execution. Any variables set are exposed as **decision signals** for the pipeline to interpret.

| Category             | Tools                                               | Description                       |
| -------------------- | --------------------------------------------------- | --------------------------------- |
| **File System**      | `read_file`, `list_directory`                       | File and directory operations     |
| **Git Operations**   | `get_commit_info`, `get_pull_request_info`, `get_repository_info`, `get_branch_policy`         | Git repository analysis           |
| **Pipeline Context** | `get_pipeline_variables`, `set_pipeline_variable`, `get_pipeline_timeline`   | Azure DevOps pipeline context |
| **Build Context**      | `get_build_context`, `get_build_changes`, `get_build_info`, `get_test_results`, `check_artifact_exists`, `get_build_work_items` | Build context         |
| **Notifications**    | `send_notification`            | Team communication                |
| **Environment**      | `get_environment_variable`      | Environment validation            |

> **Note:** These tools are designed to **observe and extract context**, not to execute builds, tests, or deployments. Any decisions derived from these signals are surfaced as pipeline variables, leaving execution behavior fully under pipeline control.

Extensibility
-------------

Smart Task is designed to support richer decision-making by allowing **additional context** to be supplied at runtime.

### Current Extensibility Model

Smart Task does **not** support registering new executable tools at this time. Instead, extensibility is achieved via the **additionalContext input parameter** of the task.

You can use this parameter to provide structured data such as:

*   Organizational policies and decision criteria
    
*   Cost and compliance signals
    
*   Service or product metadata
    
*   Outputs from previous pipeline steps
    
*   Knowledge base summaries or structured context

This additional context becomes part of the decision workflow’s input and can influence how decision signals are resolved.

### Benefits of This Approach

*   Keeps pipeline logic **safe and read-only**
    
*   Preserves execution **determinism**
    
*   Makes decision inputs **explicit and auditable**
    
*   Allows organizations to evolve decision context without modifying pipeline YAML

Installation
------------

1. Clone this repository:

```bash
git clone https://github.com/pratikxpanda/smart-task-azure-devops-pipelines.git
cd smart-task-azure-devops-pipelines
npm install
npm run build
```

2. Package and install in Azure DevOps:

```bash
npm run package
# Upload the generated .vsix file to your Azure DevOps organization
```

Configuration
-------------

Set these as Azure DevOps pipeline variables or in a variable group:

| Variable                       | Description                     | Required | Example                   |
| ------------------------------ | ------------------------------- | -------- | ------------------------- |
| `MODEL_TYPE`                   | AI model type                   | Yes      | `AZURE_OPENAI`            |
| `AZURE_OPENAI_INSTANCE_NAME`   | Your Azure OpenAI instance name | Yes      | `my-openai-instance`      |
| `AZURE_OPENAI_KEY`             | API key for authentication      | Yes      | `$(AZURE_OPENAI_API_KEY)` |
| `AZURE_OPENAI_DEPLOYMENT_NAME` | Model deployment name           | Yes      | `gpt-4o`                  |
| `AZURE_OPENAI_API_VERSION`     | API version                     | Yes      | `2025-01-01-preview`      |

Usage
-----

Smart Task runs as a **pipeline task** and emits decision signals as pipeline variables.

Example 1: Deployment Risk Assessment and Strategy Recommendation
-----------------------------------------------------------------

```yaml
- task: SmartTask@1
  displayName: 'AI Decision: Deployment Risk Assessment'
  env:
    MODEL_TYPE: $(MODEL_TYPE)
    AZURE_OPENAI_INSTANCE_NAME: $(AZURE_OPENAI_INSTANCE_NAME)
    AZURE_OPENAI_KEY: $(AZURE_OPENAI_KEY)
    AZURE_OPENAI_DEPLOYMENT_NAME: $(AZURE_OPENAI_DEPLOYMENT_NAME)
    AZURE_OPENAI_API_VERSION: $(AZURE_OPENAI_API_VERSION)
  inputs:
    prompt: |
      Analyze the current deployment candidate and assess the deployment risk.

      1. Evaluate the nature of code changes included since the last production release
      2. Identify changes to business-critical logic, data models, or external integrations
      3. Consider rollback complexity and blast radius
      4. Factor in the target environment and service criticality

      Set the following decision signals:
      - DEPLOYMENT_RISK (low | medium | high)
      - RECOMMENDED_DEPLOYMENT_STRATEGY (rolling | canary | blue-green)
      - CONFIDENCE
      - REASONING

      Base decisions on real change impact rather than branch name alone.
    additionalContext: |
      {
        "service_context": {
          "service_type": "payment_service",
          "criticality": "high",
          "supports_canary": true,
          "supports_blue_green": true
        },
        "deployment_context": {
          "environment": "production",
          "release_model": "promotion_based",
          "rollback_strategy": "automated"
        },
        "risk_guidelines": {
          "schema_changes": "high risk",
          "business_logic_changes": "high risk",
          "config_only_changes": "low risk",
          "observability_changes": "low risk"
        }
      }

```

Example 2: Intelligent Security Scanning
---------------------------------------

```yaml
- task: SmartTask@1
  displayName: 'AI Decision: Security Scanning Strategy'
  env:
    MODEL_TYPE: $(MODEL_TYPE)
    AZURE_OPENAI_INSTANCE_NAME: $(AZURE_OPENAI_INSTANCE_NAME)
    AZURE_OPENAI_KEY: $(AZURE_OPENAI_KEY)
    AZURE_OPENAI_DEPLOYMENT_NAME: $(AZURE_OPENAI_DEPLOYMENT_NAME)
    AZURE_OPENAI_API_VERSION: $(AZURE_OPENAI_API_VERSION)
  inputs:
    prompt: |
      Determine security scanning strategy based on code changes and deployment context:

      1. Analyze recent code changes to identify security-relevant modifications
      2. Check if dependencies have been updated or added
      3. Evaluate authentication/authorization code changes
      4. Consider the target deployment environment and compliance requirements

      Set the following decision signals:
      - SECURITY_SCAN_TYPE
      - SCAN_SCOPE
      - COMPLIANCE_CHECK
      - CONFIDENCE
      - REASONING

      Optimize scanning time while ensuring adequate security coverage.
    additionalContext: |
      {
        "deployment_context": {
          "target_environment": "$(DEPLOYMENT_ENVIRONMENT)",
          "compliance_requirements": ["SOC2", "GDPR"],
          "security_baseline": "enterprise"
        },
        "scanning_strategies": {
          "full_scan": "comprehensive security analysis - use for production deployments",
          "incremental_scan": "focus on changed files and dependencies - use for development",
          "dependency_scan": "focus on new/updated dependencies - use for dependency updates",
          "compliance_scan": "focus on regulatory compliance - use for production releases"
        },
        "risk_factors": {
          "production_deployment": "high risk - requires comprehensive scanning",
          "dependency_changes": "medium risk - focus on supply chain security",
          "auth_changes": "high risk - requires authentication security review"
        }
      }

```

Example 3: Intelligent Test Strategy
------------------------------------

```yaml
- task: SmartTask@1
  name: SmartDecision
  displayName: 'AI Decision: Intelligent Test Strategy'
  env:
    MODEL_TYPE: $(MODEL_TYPE)
    AZURE_OPENAI_INSTANCE_NAME: $(AZURE_OPENAI_INSTANCE_NAME)
    AZURE_OPENAI_KEY: $(AZURE_OPENAI_KEY)
    AZURE_OPENAI_DEPLOYMENT_NAME: $(AZURE_OPENAI_DEPLOYMENT_NAME)
    AZURE_OPENAI_API_VERSION: $(AZURE_OPENAI_API_VERSION)
  inputs:
    prompt: |
      Analyze this repository and the current build context to make intelligent testing decisions:

      1. Analyze the codebase structure:
         - Examine package.json to understand the project type and dependencies
         - Determine if this is a frontend, backend, or full-stack project
         - Identify testing frameworks and scripts available

      2. Analyze the build context:
         - Check the source branch to understand the type of changes (feature, hotfix, main/master)
         - Consider the build trigger reason (manual, pull request, scheduled, etc.)
         - Evaluate the scope and risk level of changes

      3. Make intelligent testing decisions:
         - For main/master branch: Run comprehensive test suite for maximum confidence
         - For feature branches: Run targeted tests based on likely impact areas
         - For hotfix branches: Focus on critical path testing for fast feedback
         - For dependency updates: Emphasize integration and compatibility tests

      4. Set pipeline variables for the test execution strategy:
         - RUN_UNIT_TESTS: true/false (for build stage)
         - RUN_INTEGRATION_TESTS: true/false (for deploy stage)
         - RUN_E2E_TESTS: true/false (for deploy stage)
         - TEST_SCOPE: 'minimal' | 'standard' | 'comprehensive'
         - TEST_REASON: clear explanation of the testing strategy decision

      Make data-driven decisions to optimize test execution time while maintaining appropriate quality gates.

    additionalContext: |
      {
        "project_context": {
          "type": "web_application",
          "environments": ["development", "staging", "production"],
          "test_types": ["unit", "integration", "e2e", "performance"],
          "pipeline_stages": ["build", "deploy"]
        },
        "decision_criteria": {
          "main_branch": "comprehensive testing required",
          "feature_branch": "targeted testing based on changes",
          "hotfix_branch": "critical path testing only",
          "scheduled_build": "full regression testing"
        },
        "stage_strategy": {
          "build_stage": "fast feedback with unit tests",
          "deploy_stage": "comprehensive validation with integration and e2e tests"
        }
      }
```

Risks and Limitations
---------------------

*   Decision quality depends on available context and prompt clarity
    
*   Not a replacement for human judgment or governance
    
*   Best introduced incrementally via pilot use cases

Development
-----------

### Setup

```bash
git clone https://github.com/pratikxpanda/smart-task-azure-devops-pipelines.git
cd smart-task-azure-devops-pipelines
npm install
npm run build
```

### Testing

```bash
# Run tests
npm test

# Local testing with development script
node scripts/dev-test.js "Your prompt here" execution
```

### Scripts

| Command           | Description              |
| ----------------- | ------------------------ |
| `npm run build`   | Compile TypeScript       |
| `npm test`        | Run tests                |
| `npm run lint`    | Run linter               |
| `npm run package` | Create extension package |

Contributing
------------

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/amazing-feature`
3. Make your changes and add tests
4. Run tests: `npm test`
5. Run linting: `npm run lint:fix`
6. Commit your changes: `git commit -m 'Add amazing feature'`
7. Push to the branch: `git push origin feature/amazing-feature`
8. Open a Pull Request

License
-------

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

**Made by [Pratik Panda](https://github.com/pratikxpanda)**
