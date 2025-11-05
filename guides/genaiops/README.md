# Azure AI Foundry GenAIOps

## Infrastructure

This architecture uses AI Foundry Agent Service as a runtime to host and serve the agents. The environments (Dev, Staging, Prod) can be separated in two different ways:

### Separate Project in Separate Foundry Resource

In this approach, for each environment a new foundry resource is deployed with a foundry project.

> **Note**: Limitations to be documented (TBC)

### Separate Project within the Same Foundry Resource

In this approach, each environment is deployed in the same foundry resource in different projects.

> **Note**: Limitations to be documented (TBC)

Agent Service runtime supports multiple artifacts related to agent development, with more planned to be released. This current design includes the following resources:

- Tools
- Memory Store
- Agents

## Workflow

The workflow is designed to run on GitHub Actions. The following approach is proposed based on a GitFlow-type workflow. This can be easily modified to implement GitHub Flow or Trunk-Based Development style workflows.

According to GitFlow convention, we have three branches in this setup: feature branches, dev branch, and main branch.

![genaiops workflow](../images/genaiops.png)

### Workflow Components

1. **Continuous Integration**: This pipeline will build the code and run tests and evaluations. Evaluation runs can be reserved for only Dev and Prod environments.

1. **Continuous Deployment**: This pipeline will deploy the artifacts to Foundry Agent Service.

### Workflow Steps

1. Whenever new code is committed to the feature branch, it triggers the `feature_ci_pipeline`. As expected, the CI pipeline runs linting checks and unit tests. However, running agent evaluations may not be useful as each commit may not have significant enough changes to produce meaningful evaluation results. Additionally, running evaluations may be slow and expensive for each commit.

1. If the CI is successful, the code is deployed to the Dev environment.

1. Once the developer is satisfied, they create a PR to the dev branch. This will trigger the `ci_pipeline`.

1. This CI pipeline runs linting, unit tests, and evaluations. If successful, the changes are merged to the dev branch.

1. The CD pipeline then deploys the agent to the dev environment for further testing and validation.

1. If the developer is satisfied with the dev environment testing, they create a PR to the main branch. This will trigger the `ci_pipeline` for production readiness.

1. This CI pipeline runs linting, unit tests, and comprehensive evaluations to ensure production quality. If all checks pass, the changes are merged to the main branch.

1. The CD pipeline deploys the agent to the production environment, completing the deployment workflow.



