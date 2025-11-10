# Azure AI Foundry GenAIOps

## Infrastructure

This architecture uses AI Foundry Agent Service as a runtime to host and serve the agents. The environments (Dev, Staging, Prod) can be separated in two different ways:

### Separate Project in Separate Foundry Resource

In this approach, for each environment a new foundry resource is deployed with a foundry project.

In-line with Microsoft guidance: "Establish distinct environments for development, testing, and production using separate resource groups or subscriptions and AI Foundry resources to isolate workflows and manage access." [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/planning)

#### Pros

1. Strong isolation boundary (blast radius limited to a single environment resource)
1. Dedicated Azure quotas & throttling limits per environment reduce noisy-neighbor risks
1. Independent RBAC scope enables least-privilege assignments and cleaner separation of duties
1. Clear cost attribution (each Foundry resource can map to a subscription, resource group, or tag set for chargeback/showback)
1. Enables divergent compliance postures (e.g., Prod with stricter policies, Dev more permissive) without complex conditional configuration
1. Simplifies disaster recovery testing (can tear down and recreate Dev without impacting Prod)
1. Enables staggered version upgrades of Foundry features or agent service runtime per environment
1. Network segmentation (private endpoints, vNET integration, firewall rules) can differ by environment for defense-in-depth
1. Reduced risk of accidental cross-environment data or secret reuse

#### Cons

1. Higher baseline cost (duplicate core Foundry resources, networking, monitoring, storage overhead)
1. Increased management overhead (more resources to provision and monitor)
1. More complex promotion workflow (artifact & model version replication between resources/projects)
1. Potential fragmentation of shared assets (tools, connections, embeddings, model registries) requiring sync automation
1. Requires consistent governance automation to avoid configuration drift across multiple resources
1. Cross-resource observability correlation (logs/traces/metrics) needs aggregation layer (e.g., Azure Monitor workspace union)

#### Limitations / Considerations

1. Direct intra-resource sharing of models, connections, or assets is not native; promotion requires export/import or pipeline actions
1. Global quotas are split; hitting subscription-level limits still impacts all environments
1. Duplicate networking increases private endpoint count quotas and potential address space pressure
1. More IAM role assignments can approach Azure RBAC object limits at scale
1. Latency differences may appear if environments reside in different regions (ensure alignment for performance testing fidelity)

### Separate Project within the Same Foundry Resource

In this approach, each environment is deployed in the same foundry resource in different projects.

Not in-line with Microsoft guidance: "Establish distinct environments for development, testing, and production using separate resource groups or subscriptions and AI Foundry resources to isolate workflows and manage access." [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/planning)

#### Pros

1. Lower cost footprint (shared Foundry control plane, networking, monitoring, and base storage)
1. Faster environment creation (project-level setup vs full resource provisioning)
1. Simplified artifact promotion (copy/move or version referencing within same resource)
1. Shared model registry and connections reduce duplication and synchronization complexity
1. Unified observability context (logs/metrics/traces) eases correlation across environments
1. Easier global policy enforcement (single resource boundary for Azure Policy & Defender onboarding)
1. Streamlined automation (fewer resource graphs to traverse for pipelines & GitOps)
1. Reduced secrets management surface (centralized vault / key configuration)

#### Cons

1. Weaker isolation—misconfiguration or runtime fault can have broader impact across all projects
1. Shared quotas can lead to contention (high load in Dev may degrade Prod if not governed)
1. Harder cost segmentation (requires accurate tagging + project-level usage attribution)
1. Risk of accidental cross-environment resource referencing (e.g., using Dev tool in Prod) without strict naming & policy guardrails
1. Scaling limits (projects may compete for the same compute capacity or rate limits)
1. Compliance divergence is harder (cannot easily enforce distinct policies requiring resource-level scoping)
1. Performance tests may skew results if background workloads from other environments run concurrently
1. Incident response blast radius larger (forensic containment more complex)

#### Limitations / Considerations

1. Project-level RBAC granularity may be insufficient for strict separation-of-duties in regulated workloads—validate role boundaries
1. Quota partitioning is logical only; no hard reservations per project—implement governance & alerting for usage caps
1. Network configuration is shared (no per-project vNET)—must rely on naming, tags, and policy for logical segmentation
1. Global secrets or connections could become over-permissioned and we should always enforce principle of least privilege & periodic review
1. Single point of failure: resource-wide service disruption impacts all environments (evaluate SLA & redundancy strategy)
1. Audit trails require strong project metadata tagging to preserve environment lineage for promotions and rollbacks
1. Scaling large multi-team usage may eventually justify migration to separate resources (plan an exit strategy early)


## Agent Service

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



