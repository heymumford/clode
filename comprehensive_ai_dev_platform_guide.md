# Comprehensive Guide: AI-Powered Development Platform on Azure

## Executive Summary

This document outlines a comprehensive, phased approach to building an AI-powered development platform on Azure. The platform leverages a council of specialized Large Language Models (LLMs) to automate and assist in various stages of the software development lifecycle, from planning and test generation to code creation, review, and refinement. The core components include Azure AI Studio for hosting LLM personas, LiteLLM for routing requests, Azure Container Apps for hosting LiteLLM, and GitHub Actions for orchestrating the Test-Driven Development (TDD) engine. The solution emphasizes security through VNet integration and private endpoints, cost management via resource scaling and monitoring, and operational excellence through comprehensive monitoring and automation. This guide details the infrastructure setup, AI model deployment, TDD workflow automation, polyglot capabilities, operational runbooks, cost considerations, and ethical guidelines. The goal is to create a robust, scalable, and efficient platform that significantly accelerates development while maintaining high standards of code quality and security.

## Infrastructure as Code (IaC)

The foundation of this platform is built upon Azure services, provisioned and managed using Infrastructure as Code principles, specifically with Azure Bicep. This ensures repeatable, consistent, and version-controlled deployments.

The core IaC components are detailed in **Phase 1: Azure Foundation Setup**, specifically under section **5. Conceptual Bicep Script Outlines**. These outlines include:
*   **`main.bicep`**: Orchestrates the deployment of resource groups and modules.
*   **`modules/network.bicep`**: Defines the virtual network, subnets, NSGs, and private DNS zones.
*   **`modules/aistudio.bicep`**: Sets up Azure AI Studio (Azure Machine Learning workspace) and its dependent resources like Storage, Key Vault, and Application Insights, including private endpoint configurations.
*   **`modules/containerapps_env.bicep`**: Deploys the Azure Container Apps Environment, configured for VNet integration and connection to Log Analytics.
*   **`modules/keyvault.bicep`**: Manages Azure Key Vault deployment, including private endpoint setup for secure secret management.

These Bicep modules establish a secure and scalable Azure environment with VNet isolation for all key components, including AI Studio, Container Apps, and GPU VMs. Private Endpoints are used extensively to ensure that PaaS services are not exposed to the public internet.

Refer to [Phase 1: Azure Foundation Setup](#phase-1-azure-foundation-setup) for the detailed Bicep script outlines and resource organization.

## AI Persona Prompting Guide

Effective prompting is crucial for harnessing the power of the AI Council. Each persona has a specialized role, and prompts should be tailored accordingly. The general principles include: clarity, context, role definition, and expected output format (preferably JSON or structured text).

Detailed example prompts for each persona in a practical scenario (User Profile Display feature) are provided in **Phase 4: Polyglot Code Generation Demonstration**, under section **2. Example Prompts for the AI Council**. This includes specific prompts for:
*   **Guardian (Test Generation):** Separate prompts for Python/Pytest (backend) and TypeScript/Jest (frontend).
*   **Builder (Code Generation):** Prompts for Python/FastAPI and TypeScript/React, guided by tests from Guardian.
*   **Artisan (Code Refinement):** Prompts for refining code based on test failures or for general quality improvements.

Key considerations for prompting, as detailed in **Phase 3: The TDD Engine** under **3. Explanations for Key Aspects - Prompt Engineering**, include:
*   **Pacer:** Clear feature descriptions, target languages, structured output for test plans.
*   **Maven:** Test plans as input to gather relevant context (code, docs).
*   **Builder (Tests):** Specific test case descriptions, testing framework guidance.
*   **Builder (Code):** Test files as input, adherence to project conventions.
*   **Artisan:** Failing test output, relevant code, focus on fixing errors or optimization.
*   **Guardian (Review):** Generated code and tests, review criteria (quality, security, best practices).

Consistently using few-shot prompting (providing examples within the prompt) and clearly defining expected output formats will significantly improve the reliability and utility of the LLM responses.

Refer to [Phase 4: Polyglot Code Generation Demonstration](#phase-4-polyglot-code-generation-demonstration) for specific prompt examples and [Phase 3: The TDD Engine](#phase-3-the-tdd-engine) for general prompt engineering strategies.

## GitHub Actions Workflows

The automation of the TDD process and other CI/CD tasks is managed by GitHub Actions. The primary workflow, `tdd_pipeline.yml`, orchestrates the interaction between the developer (providing a feature description) and the AI Council.

The full conceptual YAML for the `tdd_pipeline.yml` is provided in **Phase 3: The TDD Engine**, under section **2. GitHub Actions Workflow: `tdd_pipeline.yml`**.

Key features of this workflow include:
*   **Triggers:** `workflow_dispatch` (manual trigger with feature description) and `pull_request`.
*   **Job Structure:**
    *   `generate_tests`: Pacer plans, Maven gathers context, Builder generates tests.
    *   `generate_code`: Builder generates application code to pass the tests.
    *   `run_tests`: Executes tests for polyglot environments (Python, Node.js, Java as examples).
    *   `refine_or_review`: Artisan refines code if tests fail; Guardian reviews code if tests pass.
*   **Mocked Interactions:** The provided YAML uses `echo` statements and file manipulations to simulate LLM calls. Real implementation requires scripts to interact with the LiteLLM API.
*   **State Management:** Uses artifacts to pass data between jobs (in mock); real implementation would commit changes to a PR branch.
*   **Secret Management:** Uses GitHub secrets for API keys and endpoints.
*   **Polyglot Support:** Demonstrates matrix strategy for running tests across multiple languages.

This workflow is designed to be modular and extensible. Actual implementation of the "[MOCK]" steps involves scripting HTTP requests to the LiteLLM service, parsing responses, and managing files within the repository.

Refer to [Phase 3: The TDD Engine](#phase-3-the-tdd-engine) for the complete workflow YAML and detailed explanations of its components and operational aspects. Integration of linters and formatters within this workflow is discussed in [Phase 4: Polyglot Code Generation Demonstration](#phase-4-polyglot-code-generation-demonstration) under section **4. Managing Linters and Formatters in the TDD Workflow**.

## Architectural Diagrams (Mermaid)

Visualizing the architecture helps in understanding the relationships and data flow between components.

### High-Level System Architecture

This diagram shows the overall interaction between the user, GitHub, the AI Council, and Azure services.

```mermaid
graph TD
    A[Developer] -- 1. Pushes code / Dispatches workflow with Feature Description --> B(GitHub Actions - TDD Engine)
    B -- 2. Sends prompts to AI Council --> C{LiteLLM Proxy (Azure Container App)}
    C -- 3. Routes requests to specific LLM --> D1[Pacer (Mistral-7B on Azure AI Studio EP)]
    C -- 3. Routes requests to specific LLM --> D2[Builder (Llama-3-8B on Azure AI Studio EP)]
    C -- 3. Routes requests to specific LLM --> D3[Maven (Phi-3-mini on Azure AI Studio EP)]
    C -- 3. Routes requests to specific LLM --> D4[Guardian (CodeLlama-7b on Azure AI Studio EP)]
    C -- 3. Routes requests to specific LLM --> D5[Artisan (CodeGemma-7b on Azure AI Studio EP)]
    D1 -- 4. Returns Plan/Response --> C
    D2 -- 4. Returns Code/Response --> C
    D3 -- 4. Returns Context/Response --> C
    D4 -- 4. Returns Review/Tests --> C
    D5 -- 4. Returns Refined Code --> C
    C -- 5. Returns LLM output to Workflow --> B
    B -- 6. Commits Tests/Code/Review to PR --> G[GitHub Repository]
    A -- 7. Reviews PR --> G

    subgraph Azure VNet
        C
        D1
        D2
        D3
        D4
        D5
        E[Azure Key Vault for Secrets]
        F[Log Analytics Workspace]
    end

    C -.-> E
    D1 -.-> E
    D2 -.-> E
    D3 -.-> E
    D4 -.-> E
    D5 -.-> E
    C --> F
    D1 --> F
    D2 --> F
    D3 --> F
    D4 --> F
    D5 --> F

    style B fill:#D6EAF8,stroke:#333,stroke-width:2px
    style C fill:#D1F2EB,stroke:#333,stroke-width:2px
    style G fill:#FADBD8,stroke:#333,stroke-width:2px
```

### Network Architecture within Azure

This diagram focuses on the Azure networking setup, emphasizing VNet integration and private endpoints.

```mermaid
graph TD
    subgraph "Azure Cloud"
        subgraph "Resource Group: rg-core-infra"
            VNet[Virtual Network (vnet-main)]
            KV[Azure Key Vault]
            PrivDNS[Private DNS Zones]
            VNet -- links --> PrivDNS
            Bastion[Azure Bastion (Optional)]
            Bastion -- Peering --> VNet
        end

        subgraph "Resource Group: rg-app-hosting"
            CAE[Container Apps Environment (cae-main)]
            LiteLLM_App[LiteLLM Container App]
            CAE -- Subnet Delegation --> VNetS1[snet-containerapps]
            LiteLLM_App -- Ingress (Internal/External) --> Internet/VNetUser
            CAE -- Logs --> LAW[Log Analytics Workspace]
            LiteLLM_App -- Uses --> KV_PE[Private Endpoint for Key Vault]
        end

        subgraph "Resource Group: rg-ai-services"
            AIStudio[Azure AI Studio / ML Workspace]
            ModelEndpoint1[LLM Persona 1 EP]
            ModelEndpoint2[LLM Persona 2 EP]
            AIStudio -- Uses --> StorageAC[Storage Account]
            AIStudio -- Uses --> ACR[Container Registry]
            AIStudio -- Logs --> LAW
            ModelEndpoint1 -- Endpoint in --> VNetS2[snet-aistudio or snet-privateendpoints]
            ModelEndpoint2 -- Endpoint in --> VNetS2
            StorageAC_PE[Private Endpoint for Storage] -- Connects to --> StorageAC
            ACR_PE[Private Endpoint for ACR] -- Connects to --> ACR
            AIStudio_PE[Private Endpoint for AI Studio API] -- Connects to --> AIStudio
        end
        
        VNet --- VNetS1
        VNet --- VNetS2
        KV_PE -- Connects to --> KV
        KV_PE -- In Subnet --> VNetS2
        StorageAC_PE -- In Subnet --> VNetS2
        ACR_PE -- In Subnet --> VNetS2
        AIStudio_PE -- In Subnet --> VNetS2

    end

    Internet[Internet Users/Developers] -- HTTPS (Optional) --> LiteLLM_App
    VNetUser[Services within VNet] -- HTTP/S --> LiteLLM_App
    DevSecure[Secure Developer Access] -- HTTPS via Bastion --> Bastion
    GitHubActions[GitHub Actions Runners] -- HTTPS (via Public IP or VNet Integration for Self-hosted) --> LiteLLM_App

    style VNet fill:#AED6F1,stroke:#333,stroke-width:2px
    style CAE fill:#A9DFBF,stroke:#333,stroke-width:2px
    style AIStudio fill:#F9E79F,stroke:#333,stroke-width:2px
```

## Cost Analysis

Managing costs is paramount for the long-term viability of this AI-powered platform. The primary cost drivers will be:
1.  **GPU Compute for LLM Endpoints:** Azure Machine Learning Managed Online Endpoints (NCasT4_v3, NC6s_v3 series VMs).
2.  **Azure Container Apps:** For hosting LiteLLM (Consumption or Dedicated plan).
3.  **Azure AI Studio / ML Workspace:** Storage, potentially Application Insights, Container Registry.
4.  **GitHub Actions:** Costs for GitHub-hosted runners (minutes) or compute/maintenance for self-hosted runners.
5.  **Other Azure Services:** Key Vault, Log Analytics, Bandwidth, Storage.

**Key Cost Management Strategies (detailed in Phase 5):**
*   **Scale-to-Zero:** Configure all Azure ML managed endpoints for LLM personas to scale to zero instances when idle. This is critical for GPU cost savings.
*   **Right-Sizing Instances:** Start with smaller recommended GPU SKUs (e.g., `Standard_NC4as_T4_v3`) and monitor performance (GPU utilization, latency) before scaling up.
*   **LiteLLM Caching:** Implement caching in LiteLLM to reduce redundant calls to LLM endpoints, saving on token costs and endpoint compute.
*   **Azure Budgets & Alerts:** Set up budgets in Azure Cost Management for the overall subscription and key resource groups. Configure alerts for when costs approach thresholds.
*   **Tagging:** Consistently tag all resources for granular cost tracking.
*   **Azure Advisor:** Regularly review Azure Advisor recommendations for cost optimization.
*   **Log Retention Policies:** Configure appropriate retention policies for Log Analytics to manage storage costs.
*   **GitHub Actions Optimization:** Use caching for dependencies, optimize workflow triggers, and consider self-hosted runners if GPU tasks are brought into the workflow directly (though current design uses Azure ML for GPUs).

A detailed breakdown of cost management approaches is available in **Phase 5: Autonomous Operation and Cost Management**, sections **2. Cost Management Strategy (Azure Cost Management + Billing)** and **3. GPU Scaling/Shutdown Automation Strategy**.

## Operational Runbook

This section outlines common operational tasks and procedures for maintaining the platform. Much of this is covered in **Phase 5: Autonomous Operation and Cost Management**.

**I. Monitoring & Alerting:**
*   **Dashboards:** Regularly review Azure Dashboards configured with key metrics (LiteLLM performance, AI Persona health, costs).
*   **Alert Response:**
    *   **High Error Rate (LiteLLM/AI Personas):** Investigate logs in Log Analytics. Check LiteLLM configuration, downstream LLM health, API key validity.
    *   **High Latency:** Check GPU/CPU utilization of endpoints. May require scaling up instances or optimizing models/prompts.
    *   **Endpoint Unavailability:** Check Azure ML Studio for endpoint status. Review deployment logs.
    *   **Budget Alerts:** Analyze spending in Azure Cost Management. Identify unexpected cost drivers and take corrective action (e.g., shutdown unused resources, optimize queries).
    *   **GitHub Actions Failure:** Review workflow logs in GitHub. Debug scripts, check secret configurations, LLM availability.

**II. LLM Endpoint Management (Azure AI Studio):**
*   **Updating Models:**
    1.  Deploy new model version as a new deployment under the same endpoint (blue/green).
    2.  Test the new deployment.
    3.  Gradually shift traffic to the new deployment.
    4.  Update LiteLLM configuration if endpoint URI/key changes.
*   **Checking Scale-to-Zero:** Verify in Azure Monitor that instance counts drop to 0 after idle period.
*   **Troubleshooting:** Use "Test" tab in endpoint details, check logs in Log Analytics.

**III. LiteLLM Management (Azure Container App):**
*   **Updating LiteLLM Version/Config:**
    1.  Build new Docker image with updated `litellm_config.yaml` or LiteLLM version.
    2.  Push to ACR.
    3.  Update Container App to use the new image tag.
    4.  Monitor logs for successful startup and test requests.
*   **Managing API Keys (via Key Vault):** Update secrets in Key Vault. Container App will pick up changes on next restart/revision or as configured.
*   **Scaling:** Adjust min/max replicas and scaling rules in Container App configuration based on load.

**IV. GitHub Actions Workflow Management:**
*   **Updating Workflow Logic (`tdd_pipeline.yml`):** Modify YAML, test in a feature branch.
*   **Managing Secrets:** Update `LITELLM_API_ENDPOINT`, `LITELLM_API_KEY`, etc., in GitHub repository secrets.
*   **Troubleshooting Failing Actions:** Examine workflow logs for specific error messages. Enable debug logging in actions if necessary.

**V. Cost Optimization Tasks:**
*   **Monthly Review:** Perform deep dive into Azure Cost Management.
*   **Review Azure Advisor:** Implement cost-saving recommendations.
*   **Identify Underutilized Resources:** Scale down or delete unused endpoints/VMs.

**VI. Security & Compliance:**
*   **Review Key Vault Access Policies:** Ensure least privilege.
*   **Monitor for Security Alerts:** In Azure Security Center.
*   **Regularly Update Dependencies:** In LiteLLM Docker image and any custom scripts.

Refer to [Phase 5: Autonomous Operation and Cost Management](#phase-5-autonomous-operation-and-cost-management) for more details on monitoring, cost management, and scaling automation.

## Nuances & Contextual Sensitivities

*   **LLM Reliability & Hallucinations:** LLMs can occasionally produce incorrect, biased, or nonsensical outputs ("hallucinations"). The TDD approach (generating tests first) helps mitigate this for code generation, as code must pass tests. For planning (Pacer) or review (Guardian), outputs should be treated as suggestions requiring human validation.
*   **Prompt Brittleness:** LLM outputs can be highly sensitive to small changes in prompts. Extensive prompt engineering and testing are required. Version control prompts.
*   **Context Window Limits:** Each LLM has a maximum context window. For tasks requiring large amounts of input (e.g., reviewing entire codebases with Maven), strategies like chunking, summarization, or using models with larger context windows (like Phi-3-mini-128k) are necessary.
*   **Token Costs:** API calls to LLMs are typically priced per token (input and output). Complex prompts or verbose outputs increase costs. Optimize prompts for conciseness while retaining clarity. LiteLLM caching is crucial here.
*   **Cold Starts:** Scale-to-zero for GPU endpoints saves costs but introduces latency for the first request after an idle period. This trade-off must be acceptable for the use case. For user-facing interactive scenarios, consider keeping a minimum of 1 instance.
*   **Data Security & Privacy with LLMs:** If proprietary code or sensitive data is included in prompts, ensure the LLM provider's terms of service and data handling policies meet your security requirements. Using Azure OpenAI Service within your Azure subscription or self-hosted models can provide greater control. The current design uses models from Azure AI Studio catalog, which are managed by Microsoft, but their data handling policies for inference should be reviewed.
*   **Polyglot Complexity:** Supporting multiple programming languages increases the complexity of prompt engineering, test execution environments, and linting/formatting toolchains. Start with one or two core languages and expand gradually.
*   **Iterative Development of the AI System:** The AI Council itself is a system that will require iteration. Monitor the performance of each persona and refine prompts, models, or even roles over time.
*   **Human Oversight is Non-Negotiable:** This system is an advanced assistant, not a replacement for human developers. All AI-generated artifacts (tests, code, reviews) must be critically reviewed by humans before merging into production.
*   **Dependency on External Services:** The system relies on Azure services and potentially GitHub. Outages or changes in these services can impact the platform.

## Ethical and Governance Considerations

*   **Bias in LLMs:** LLMs are trained on vast datasets and can inherit biases present in that data. This can manifest in generated code, tests, or review comments.
    *   **Mitigation:** Diverse testing of LLM outputs, human review to catch biased suggestions, ongoing research into bias detection and mitigation in LLMs. Be particularly mindful if generating user-facing text or making decisions that could impact users.
*   **Accountability & Responsibility:** Who is responsible if AI-generated code introduces a bug or security vulnerability?
    *   **Framework:** Ultimately, the human developer and the organization deploying the code are responsible. The AI is a tool. Clear policies should state that human review and approval are mandatory.
*   **Intellectual Property (IP):**
    *   **Training Data:** LLMs are trained on large amounts of code, some ofwhich may be under restrictive licenses. There's ongoing legal debate about the IP implications of models trained on public code.
    *   **Generated Code:** The ownership and licensing of AI-generated code can be ambiguous.
    *   **Mitigation:** Use models from reputable sources (like Azure AI Studio catalog) that address these concerns. Consult legal counsel regarding IP implications for your specific use case and jurisdiction. Avoid feeding highly sensitive or proprietary IP directly into prompts sent to third-party model APIs unless data privacy and usage terms are explicitly clear and acceptable.
*   **Security of AI-Generated Code:** AI might generate code with subtle vulnerabilities if not specifically prompted or guided against them.
    *   **Mitigation:** Guardian persona's role is crucial. Integrate SAST (Static Application Security Testing) tools into the CI/CD pipeline. Thorough human review of security-sensitive code.
*   **Transparency & Explainability:** LLM decision-making processes are often opaque ("black box").
    *   **Mitigation:** While full explainability is a research challenge, design prompts to ask for reasoning (e.g., "Explain why this change was made"). Focus on the functional correctness (tests passing) and human review rather than solely relying on the AI's "intent."
*   **Skill Degradation:** Over-reliance on AI for tasks like coding or test writing could potentially lead to skill degradation in developers.
    *   **Mitigation:** Frame the AI as an assistant and productivity tool, not a replacement. Encourage developers to understand, review, and learn from AI suggestions. Use it to automate boilerplate and free up developers for more complex problem-solving.
*   **Data Privacy (in Prompts):** Avoid including Personally Identifiable Information (PII) or other sensitive data in prompts sent to external LLM APIs unless absolutely necessary and permitted under strict data processing agreements.
    *   **Mitigation:** Use placeholders for sensitive data in prompts. If real data is needed, ensure the chosen LLM service (e.g., Azure OpenAI Service configured within your tenant) offers appropriate data privacy and security guarantees.
*   **Governance Framework:**
    *   Establish clear guidelines for the use of the AI development platform.
    *   Define roles and responsibilities for overseeing the AI system.
    *   Implement a process for regularly reviewing and updating ethical guidelines as AI technology evolves.
    *   Ensure compliance with relevant industry regulations and data protection laws.

By proactively addressing these nuances and ethical considerations, the AI-powered development platform can be a responsible and transformative asset.

---

# Phase 1: Azure Foundation Setup

This document outlines the foundational Azure setup for our project. It includes details on resource group structure, region selection, networking design, SKU selections for key Azure services, and conceptual Bicep scripts for deployment.

## 1. Resource Group Structure

We will adopt a modular resource group strategy to organize resources based on their lifecycle and purpose.

*   **`rg-core-infra-<env>`**: This resource group will host core infrastructure components like the virtual network (VNet), Network Security Groups (NSGs), private DNS zones, and Key Vault.
*   **`rg-ai-services-<env>`**: This resource group will contain Azure AI Studio instances, associated storage accounts, and Cognitive Services.
*   **`rg-app-hosting-<env>`**: This resource group will house Azure Container Apps environment, Container Apps, and related resources like Log Analytics workspace.
*   **`rg-gpu-compute-<env>`**: This resource group will be dedicated to GPU Virtual Machines used for intensive model training or processing.

Replace `<env>` with the environment identifier (e.g., `dev`, `staging`, `prod`).

## 2. Region Selection

*   **Primary Region**: `East US 2` - Chosen for its wide range of service availability, including latest GPU SKUs and AI services.
*   **Secondary Region (for DR)**: `Central US` - Selected for geographic separation and availability of complementary services.

All resources within an environment will initially be deployed to the primary region. Disaster Recovery strategies will leverage the secondary region.

## 3. Networking Design

### 3.1. Virtual Network (VNet)

*   **Name**: `vnet-main-<env>-<region_shortcode>` (e.g., `vnet-main-dev-eus2`)
*   **Address Space**: `10.X.0.0/16` (e.g., `10.1.0.0/16` for dev, `10.2.0.0/16` for staging)

### 3.2. Subnets

*   **`snet-containerapps-<env>-<region_shortcode>`**:
    *   Address Prefix: `10.X.1.0/24`
    *   Purpose: For Azure Container Apps Environment.
    *   Service Delegation: `Microsoft.App/environments`
*   **`snet-aistudio-<env>-<region_shortcode>`**:
    *   Address Prefix: `10.X.2.0/24`
    *   Purpose: For Azure AI Studio related resources (e.g., private endpoints for storage).
*   **`snet-gpuvms-<env>-<region_shortcode>`**:
    *   Address Prefix: `10.X.3.0/24`
    *   Purpose: For GPU Virtual Machines.
*   **`snet-privateendpoints-<env>-<region_shortcode>`**:
    *   Address Prefix: `10.X.4.0/24`
    *   Purpose: Dedicated subnet for private endpoints.
*   **`snet-appgateway-<env>-<region_shortcode>` (Optional)**:
    *   Address Prefix: `10.X.5.0/26`
    *   Purpose: If Azure Application Gateway is used for ingress.
*   **`AzureBastionSubnet` (Optional)**:
    *   Address Prefix: `10.X.6.0/27`
    *   Purpose: If Azure Bastion is used for secure VM access.

### 3.3. Network Security Groups (NSGs)

*   NSGs will be applied at the subnet level.
*   Default NSG rules will be restrictive, only allowing necessary traffic.
*   Specific rules:
    *   **`nsg-containerapps`**: Allow necessary traffic for Container Apps (e.g., HTTPS, internal VNet communication).
    *   **`nsg-aistudio`**: Allow traffic related to AI Studio operations and private endpoint communication.
    *   **`nsg-gpuvms`**: Allow SSH/RDP (if needed, ideally via Bastion) and traffic for data transfer and model training.
    *   **`nsg-privateendpoints`**: Typically allows all traffic from within the VNet but denies all from outside.

### 3.4. Private Endpoints

Private Endpoints will be used extensively to secure access to PaaS services:
*   Azure AI Studio and its associated Storage Account, Key Vault, Container Registry.
*   Azure Container Registry (if used for custom images).
*   Azure Key Vault.
*   Azure Storage Accounts used by various services.

### 3.5. DNS

*   **Private DNS Zones**:
    *   `privatelink.blob.core.windows.net`
    *   `privatelink.file.core.windows.net`
    *   `privatelink.queue.core.windows.net`
    *   `privatelink.table.core.windows.net`
    *   `privatelink.vaultcore.azure.net` (Key Vault)
    *   `privatelink.api.azureml.ms` (Azure AI Studio/Azure Machine Learning)
    *   `privatelink.<region>.inference.ml.azure.com` (Azure AI Studio/Azure Machine Learning inference)
    *   `privatelink.notebooks.azure.net` (Azure AI Studio/Azure Machine Learning notebooks)
    *   `privatelink.azurecontainerapps.io` (Azure Container Apps Environment)
*   These zones will be linked to the main VNet (`vnet-main-<env>-<region_shortcode>`).

## 4. SKU Selections

### 4.1. Azure Container Apps Environment

*   **Plan**: Consumption + Dedicated plan group.
    *   Start with Consumption for general workloads.
    *   Utilize Dedicated plan instances for workloads requiring more consistent performance or specific hardware (e.g., memory-optimized).
*   SKU choice within Dedicated plan will depend on specific app requirements (e.g., `Standard_D4_v5`).

### 4.2. Azure AI Studio

*   Azure AI Studio itself doesn't have a "SKU" in the traditional sense. It's a platform that orchestrates other Azure services.
*   **Associated Azure Machine Learning Workspace**: Standard tier.
*   **Associated Storage Account**: `Standard_LRS` or `Standard_GRS` (for DR). Consider `Premium_LRS` if blob access performance is critical for datasets.
*   **Associated Key Vault**: Standard tier.
*   **Associated Application Insights**: Basic or Standard, depending on logging and monitoring needs.
*   **Associated Container Registry**: Basic for initial development, Standard or Premium for production workloads with geo-replication needs.

### 4.3. GPU Virtual Machines

*   **Series**: NCasT4_v3-series, NC_A100_v4 series, or ND_A100_v4 series.
    *   **`NCasT4_v3-series` (e.g., `Standard_NC4as_T4_v3`)**: Good for smaller training jobs, inference, or development.
    *   **`NC_A100_v4 series` or `ND_A100_v4 series`**: For large-scale training and demanding HPC workloads. Choice depends on memory and interconnect requirements.
*   **OS Disk**: Premium SSD for better performance.
*   **Storage**: Premium SSD or Ultra Disk for datasets and model checkpoints, depending on IOPS and throughput requirements.

## 5. Conceptual Bicep Script Outlines

The following Bicep files provide a conceptual structure for deploying the foundation.

### `main.bicep`

```bicep
// main.bicep
targetScope = 'subscription' // or resourceGroup if deploying RGs manually first

param environment string = 'dev'
param location string = 'eastus2'
param vnetAddressPrefix string = '10.1.0.0/16'

var resourceGroupNameCore = 'rg-core-infra-${environment}'
var resourceGroupNameAI = 'rg-ai-services-${environment}'
var resourceGroupNameApps = 'rg-app-hosting-${environment}'
var resourceGroupNameGPU = 'rg-gpu-compute-${environment}'

// Resource Group Definitions
resource rgCore 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: resourceGroupNameCore
  location: location
}

resource rgAI 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: resourceGroupNameAI
  location: location
}

resource rgApps 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: resourceGroupNameApps
  location: location
}

resource rgGPU 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: resourceGroupNameGPU
  location: location
}

// Core Infrastructure Module
module network 'modules/network.bicep' = {
  scope: resourceGroup(resourceGroupNameCore)
  name: 'networkDeployment'
  params: {
    vnetName: 'vnet-main-${environment}-${substring(location, 0, 4)}'
    vnetAddressPrefix: vnetAddressPrefix
    location: location
    environment: environment
  }
  dependsOn: [rgCore]
}

module keyvault 'modules/keyvault.bicep' = {
  scope: resourceGroup(resourceGroupNameCore)
  name: 'keyvaultDeployment'
  params: {
    kvName: 'kv-main-${environment}-${substring(location, 0, 4)}'
    location: location
    environment: environment
    vnetId: network.outputs.vnetId
    privateEndpointSubnetId: network.outputs.privateEndpointSubnetId
  }
  dependsOn: [rgCore, network]
}

// AI Services Module
module aiStudio 'modules/aistudio.bicep' = {
  scope: resourceGroup(resourceGroupNameAI)
  name: 'aiStudioDeployment'
  params: {
    aiStudioName: 'aistudio-${environment}-${substring(location, 0, 4)}'
    location: location
    environment: environment
    // Pass necessary IDs from network and keyvault modules
    vnetId: network.outputs.vnetId 
    privateEndpointSubnetId: network.outputs.privateEndpointSubnetId
    keyVaultId: keyvault.outputs.keyVaultId
  }
  dependsOn: [rgAI, network, keyvault]
}

// App Hosting Module
module containerAppsEnv 'modules/containerapps_env.bicep' = {
  scope: resourceGroup(resourceGroupNameApps)
  name: 'containerAppsEnvDeployment'
  params: {
    caeName: 'cae-${environment}-${substring(location, 0, 4)}'
    location: location
    environment: environment
    vnetId: network.outputs.vnetId
    controlPlaneSubnetId: network.outputs.containerAppsSubnetId // Assuming specific subnet for CAE control plane if different
    appSubnetId: network.outputs.containerAppsSubnetId
    logAnalyticsWorkspaceId: null // Placeholder, ideally create/reference LA workspace
    keyVaultId: keyvault.outputs.keyVaultId
  }
  dependsOn: [rgApps, network, keyvault]
}

// GPU VMs module would be similar, defining VMs within rg-gpu-compute-<env>
// module gpuVms 'modules/gpuvms.bicep' = { ... }

output coreRgName string = resourceGroupNameCore
output aiRgName string = resourceGroupNameAI
output appsRgName string = resourceGroupNameApps
output gpuRgName string = resourceGroupNameGPU
output vnetId string = network.outputs.vnetId
```

### `modules/network.bicep`

```bicep
// modules/network.bicep
param vnetName string
param vnetAddressPrefix string
param location string
param environment string

// Define subnet configurations as an array of objects
var subnets = [
  {
    name: 'snet-containerapps-${environment}-${substring(location, 0, 4)}'
    addressPrefix: cidrSubnet(vnetAddressPrefix, 24, 1) // 10.X.1.0/24
    delegations: [
      {
        name: 'Microsoft.App.environments'
        properties: {
          serviceName: 'Microsoft.App/environments'
        }
      }
    ]
    privateEndpointNetworkPolicies: 'Disabled' // Required for CAE internal environment
    privateLinkServiceNetworkPolicies: 'Enabled' 
  }
  {
    name: 'snet-aistudio-${environment}-${substring(location, 0, 4)}'
    addressPrefix: cidrSubnet(vnetAddressPrefix, 24, 2) // 10.X.2.0/24
  }
  {
    name: 'snet-gpuvms-${environment}-${substring(location, 0, 4)}'
    addressPrefix: cidrSubnet(vnetAddressPrefix, 24, 3) // 10.X.3.0/24
  }
  {
    name: 'snet-privateendpoints-${environment}-${substring(location, 0, 4)}'
    addressPrefix: cidrSubnet(vnetAddressPrefix, 24, 4) // 10.X.4.0/24
    privateEndpointNetworkPolicies: 'Enabled' // Or 'Disabled' if issues with NSG application on PE subnet
  }
  // Add other subnets like AppGateway, Bastion if needed
]

resource vnet 'Microsoft.Network/virtualNetworks@2022-07-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        vnetAddressPrefix
      ]
    }
    subnets: [for (subnet, i) in subnets: {
      name: subnet.name
      properties: {
        addressPrefix: subnet.addressPrefix
        delegations: contains(subnet, 'delegations') ? subnet.delegations : []
        privateEndpointNetworkPolicies: contains(subnet, 'privateEndpointNetworkPolicies') ? subnet.privateEndpointNetworkPolicies : 'Enabled'
        privateLinkServiceNetworkPolicies: contains(subnet, 'privateLinkServiceNetworkPolicies') ? subnet.privateLinkServiceNetworkPolicies : 'Enabled'
        // Add NSG association here if NSGs are created in this module
      }
    }]
  }
}

// Example NSG (repeat for other subnets)
resource nsgContainerApps 'Microsoft.Network/networkSecurityGroups@2022-07-01' = {
  name: 'nsg-containerapps-${environment}-${substring(location, 0, 4)}'
  location: location
  properties: {
    securityRules: [
      // Define rules here
    ]
  }
}

// Associate NSG to subnet (example)
resource subnetContainerAppsNsgAssociation 'Microsoft.Network/virtualNetworks/subnets@2022-07-01' = {
  parent: vnet
  name: subnets[0].name // Assuming first subnet is for Container Apps
  properties: {
    networkSecurityGroup: {
      id: nsgContainerApps.id
    }
  }
}


// Private DNS Zones (example for blob)
resource privateDnsZoneBlob 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.blob.core.windows.net'
  location: 'global' // Private DNS zones are global
  properties: {}
}

resource vnetLinkBlob 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: privateDnsZoneBlob
  name: '${vnetName}-link'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: vnet.id
    }
  }
}
// ... Repeat for other required Private DNS Zones ...

output vnetId string = vnet.id
output vnetName string = vnet.name
output containerAppsSubnetId string = vnet.properties.subnets[0].id // Adjust index as per your subnets array
output privateEndpointSubnetId string = vnet.properties.subnets[3].id // Adjust index
// Output other subnet IDs as needed
```

### `modules/aistudio.bicep`

```bicep
// modules/aistudio.bicep
param aiStudioName string // This will be the Azure Machine Learning Workspace name
param location string
param environment string

param keyVaultId string
param privateEndpointSubnetId string // For AI Studio's dependent services private endpoints

// AI Studio requires: Storage, Key Vault, App Insights, Container Registry (optional but recommended)
// For simplicity, we'll show AML Workspace which is the core of AI Studio

var storageAccountName = 'st${replace(aiStudioName, '-', '')}${uniqueString(resourceGroup().id)}' // Create globally unique name
var appInsightsName = 'appi-${aiStudioName}'
var mlWorkspaceName = aiStudioName // AI Studio Project is an AML Workspace

resource storage 'Microsoft.Storage/storageAccounts@2022-09-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    // Configure for private access if possible, or use service endpoints initially
    publicNetworkAccess: 'Disabled' // Aim for this with Private Endpoints
  }
}

resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: appInsightsName
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
  }
}

resource mlWorkspace 'Microsoft.MachineLearningServices/workspaces@2023-04-01' = {
  name: mlWorkspaceName
  location: location
  sku: {
    name: 'Standard' // Or 'Enterprise' if features are needed; 'Basic' also exists
  }
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    friendlyName: mlWorkspaceName
    storageAccount: storage.id
    keyVault: keyVaultId
    applicationInsights: appInsights.id
    // hbiWorkspace: false // High Business Impact workspace
    publicNetworkAccess: 'Disabled' // Important for private setup
  }
}

// Private Endpoints for AML Workspace and its storage
// This is a simplified example; AML has multiple sub-resources for PEs (api, notebook, etc.)
resource peStorageBlob 'Microsoft.Network/privateEndpoints@2022-07-01' = {
  name: 'pe-${storageAccountName}-blob'
  location: location
  properties: {
    subnet: {
      id: privateEndpointSubnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${storageAccountName}-blob-connection'
        properties: {
          privateLinkServiceId: storage.id
          groupIds: ['blob'] // For blob storage
        }
      }
    ]
  }
}
// ... other PEs for file, queue, table for storage, and for AML workspace itself ...

output mlWorkspaceId string = mlWorkspace.id
output mlWorkspaceName string = mlWorkspace.name
```

### `modules/containerapps_env.bicep`

```bicep
// modules/containerapps_env.bicep
param caeName string
param location string
param environment string

param appSubnetId string // Subnet for the apps
// param controlPlaneSubnetId string // Subnet for the control plane (if using internal environment) - now often same as appSubnetId
param logAnalyticsWorkspaceId string // Optional, but recommended
param keyVaultId string // For secrets

// If Log Analytics Workspace is not passed, create a new one or decide on a naming convention
resource laWorkspace 'Microsoft.OperationalInsights/workspaces@2022-10-01' = if (empty(logAnalyticsWorkspaceId)) {
  name: 'la-${caeName}-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}

var actualLogAnalyticsId = !empty(logAnalyticsWorkspaceId) ? logAnalyticsWorkspaceId : laWorkspace.id

resource cae 'Microsoft.App/managedEnvironments@2023-05-01' = {
  name: caeName
  location: location
  sku: {
    name: 'Consumption' // Or 'Premium' for workload profiles environment
  }
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: reference(actualLogAnalyticsId, '2022-10-01').customerId
        sharedKey: listKeys(actualLogAnalyticsId, '2022-10-01').primarySharedKey
      }
    }
    vnetConfiguration: {
      internal: true // True for VNet integrated environment, requires specific subnet config
      infrastructureSubnetId: appSubnetId // For internal environments, this is where infra is provisioned
      // runtimeSubnetId: appSubnetId // For workload profiles based environment
    }
    zoneRedundant: false // Consider true for production in regions that support it
  }
}

// Example of a Container App within the environment
/*
resource myContainerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: 'myapp-${environment}'
  location: location
  properties: {
    managedEnvironmentId: cae.id
    configuration: {
      // Secrets can come from Key Vault via reference
      // secrets: [
      //   {
      //     name: 'mysecret'
      //     keyVaultUrl: 'kvreference_to_secret_uri'
      //     identity: 'system' // Or user-assigned managed identity
      //   }
      // ]
      ingress: {
        external: false // Internal only
        targetPort: 80 
        transport: 'http'
        // allowInsecure: false // if using http
      }
      // dapr: { enabled: true, appId: 'myapp' ... }
      // registries: [ ... ]
    }
    template: {
      containers: [
        {
          image: 'mcr.microsoft.com/azuredocs/containerapps-helloworld:latest'
          name: 'simple-hello-world'
          resources: {
            cpu: json('0.25')
            memory: '0.5Gi'
          }
        }
      ]
      scale: {
        minReplicas: 1
        maxReplicas: 3
      }
    }
  }
}
*/

output caeId string = cae.id
output caeName string = cae.name
output caeDefaultDomain string = cae.properties.defaultDomain
```

### `modules/keyvault.bicep`

```bicep
// modules/keyvault.bicep
param kvName string
param location string
param environment string

param vnetId string // Not directly used for KV creation, but for context or if NSG rules depend on it
param privateEndpointSubnetId string

// Recommended: Enable RBAC for permissions
resource keyVault 'Microsoft.KeyVault/vaults@2022-07-01' = {
  name: kvName
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'Standard'
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true // Recommended over access policies
    // softDeleteRetentionInDays: 90
    // enableSoftDelete: true
    publicNetworkAccess: 'Disabled' // Secure with Private Endpoint
  }
}

resource peKeyVault 'Microsoft.Network/privateEndpoints@2022-07-01' = {
  name: 'pe-${kvName}'
  location: location
  properties: {
    subnet: {
      id: privateEndpointSubnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${kvName}-connection'
        properties: {
          privateLinkServiceId: keyVault.id
          groupIds: ['vault'] // Group ID for Key Vault
        }
      }
    ]
  }
}

// Example: Granting an identity access to the Key Vault (e.g., an App Service or VM Managed Identity)
// param principalIdToGrantAccess string
// resource kvSecretUserRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
//   scope: keyVault
//   name: guid(keyVault.id, principalIdToGrantAccess, 'kvSecretUser')
//   properties: {
//     roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6') // Key Vault Secrets User
//     principalId: principalIdToGrantAccess
//     principalType: 'ServicePrincipal' // Or 'User', 'Group'
//   }
// }


output keyVaultId string = keyVault.id
output keyVaultUri string = keyVault.properties.vaultUri
```

This structure provides a scalable and organized way to deploy the Azure foundation using Bicep. Each module focuses on a specific area, making it easier to manage and update.
Remember to replace placeholders like `<env>` and adjust CIDR blocks, names, and SKU selections according to your specific project requirements.

---

# Phase 2: The AI Council Assembled

This phase focuses on selecting and deploying the Large Language Models (LLMs) that form the "AI Council," and setting up the LiteLLM router for unified access.

## 1. LLM Selection Rationale

The AI Council comprises specialized LLMs, each chosen for its strengths in specific areas, creating a synergistic multi-agent system. All selected models are open-source and can be fine-tuned if necessary. We are prioritizing instruction-tuned models for better out-of-the-box performance on directed tasks.

*   **Pacer (Orchestrator & Planner): Mistral-7B-Instruct-v0.2**
    *   **Rationale:** Mistral-7B is a highly capable model for its size, known for its strong reasoning, instruction following, and efficiency. The instruct version is well-suited for understanding complex prompts and breaking down tasks, making it ideal for the Pacer role which orchestrates the overall workflow and plans sub-tasks. Its relatively smaller size allows for cost-effective deployment.

*   **Builder (Code Generation & General Tasks): Llama-3-8B-Instruct**
    *   **Rationale:** Llama-3-8B-Instruct is Meta's latest generation small-to-mid-size model, demonstrating significant improvements in coding, reasoning, and general instruction following compared to its predecessors. It offers a good balance between performance and resource requirements, making it a strong candidate for the Builder role, which handles diverse tasks including code generation and general problem-solving.

*   **Maven (Knowledge & Context Expert): Phi-3-mini-128k-instruct**
    *   **Rationale:** Microsoft's Phi-3-mini-128k-instruct is a powerful small language model (SLM) with a surprisingly large context window (128k tokens). This makes it exceptionally well-suited for tasks requiring deep understanding of large documents or extensive context, such as RAG (Retrieval Augmented Generation) or acting as a knowledge expert. Its compact size also makes it efficient to deploy.

*   **Guardian (Code Review & Security): CodeLlama-7b-Instruct**
    *   **Rationale:** CodeLlama-7b-Instruct is specifically fine-tuned for code-related tasks, including code completion, debugging, and explanation. Its instruct tuning helps in guiding it to perform specific review and security analysis functions. It's a natural fit for the Guardian role, focusing on code quality, identifying potential vulnerabilities, and ensuring adherence to best practices.

*   **Artisan (Code Refinement & Optimization): CodeGemma-7b-it**
    *   **Rationale:** Google's CodeGemma-7b-it (instruction-tuned) is another strong contender in the code generation and understanding space, built upon the Gemma foundation. It's selected for the Artisan role to provide an alternative perspective on code refinement, optimization, and potentially different coding style suggestions compared to CodeLlama or Llama-3. Having diversity in code-focused models can lead to more robust and well-rounded code outputs.

## 2. Deploying LLMs as Managed Online Endpoints in Azure AI Studio

Azure AI Studio (leveraging Azure Machine Learning) allows for deploying models from its catalog (or custom models) as managed online endpoints. This provides a scalable and secure way to serve LLMs.

**Conceptual Step-by-Step Guide:**

1.  **Navigate to Azure AI Studio/Azure Machine Learning Studio.**
2.  **Access the Model Catalog:**
    *   Find the desired models (Mistral-7B-Instruct-v0.2, Llama-3-8B-Instruct, Phi-3-mini-128k-instruct, CodeLlama-7b-Instruct, CodeGemma-7b-it). Azure regularly updates its catalog. If a specific version isn't available, you might need to register a custom model from Hugging Face or another source.
3.  **Deploy to an Online Endpoint:**
    *   For each model, select the "Deploy" option and choose "Real-time endpoint."
4.  **Endpoint Configuration:**
    *   **Endpoint Name:** Define a unique name (e.g., `pacer-mistral-7b-ep`, `builder-llama-3-8b-ep`).
    *   **Deployment Name:** Define a name for this specific deployment (e.g., `blue`, `v1`).
    *   **Virtual Machine Selection:**
        *   **Recommended SKUs:** `Standard_NC4as_T4_v3` (4 vCPUs, 28 GiB RAM, 1 T4 GPU) or `Standard_NC6s_v3` (6 vCPUs, 112 GiB RAM, 1 V100 GPU).
        *   **Rationale:**
            *   `Standard_NC4as_T4_v3`: Suitable for 7B parameter models like Mistral-7B, CodeLlama-7B, CodeGemma-7b, and Phi-3-mini. It offers a good balance of performance and cost for these model sizes.
            *   `Standard_NC6s_v3`: Provides more GPU memory and compute, which can be beneficial for slightly larger models like Llama-3-8B, especially if higher throughput or lower latency is critical, or if the model's memory footprint is close to the T4's limit.
        *   Start with `Standard_NC4as_T4_v3` for the 7B models and monitor performance. Use `Standard_NC6s_v3` for Llama-3-8B or if others require more resources.
    *   **Instance Count & Scaling:**
        *   **Initial Instance Count:** Start with `1`.
        *   **Scale-to-Zero:**
            *   Enable "Scale to zero" by setting the minimum instance count to `0`.
            *   Configure the `Idle shutdown` time (e.g., 15-30 minutes). This means if the endpoint receives no requests for this duration, it will scale down to zero instances, saving costs. There will be a cold start latency when the first request arrives after scaling to zero.
        *   **Maximum Instance Count:** Set a reasonable maximum based on expected load and budget (e.g., `2` or `3` initially).
        *   **Scaling Rules (Optional):** Configure metric-based scaling (e.g., GPU utilization, CPU utilization, number of requests) for more dynamic scaling if needed beyond scale-to-zero.
    *   **Scoring Script (if needed):** For models directly from the catalog, Azure often provides a default scoring script. If deploying a custom model or needing specific input/output processing, you'll need to provide your own `score.py` and `conda.yaml` environment file.
    *   **Environment:** Select or create an environment that includes necessary dependencies (e.g., `transformers`, `torch`, `accelerate`). Models from the catalog usually have curated environments.
5.  **Networking:**
    *   Ensure the endpoint is configured to use the **private VNet** established in Phase 1, by associating it with your Azure Machine Learning workspace that is configured for VNet integration. This typically means deploying the endpoint into a workspace that has its compute and endpoints configured to reside within the VNet.
    *   Use **Private Endpoints** for accessing the scoring endpoint from within your VNet (e.g., from LiteLLM).
6.  **Security:**
    *   Authentication: Use Key-based authentication initially. The keys will be managed by LiteLLM.
    *   Managed Identity: Assign a system-assigned or user-assigned managed identity to the endpoint for accessing other Azure resources (like Key Vault for secrets in the scoring script, if needed) securely.
7.  **Review and Create:**
    *   Review all configurations and create the endpoint. Deployment can take some time (15-30 minutes or more depending on the model and VM).
8.  **Test the Endpoint:**
    *   Once deployed, use the "Test" tab in the endpoint details or use tools like `curl` or Postman with the provided scoring URI and key to send sample requests.

**Repeat steps 3-8 for each LLM in the AI Council.**

## 3. Deploying LiteLLM as an Azure Container App

LiteLLM will act as a unified proxy/router to all the deployed LLM endpoints, providing a consistent interface and managing API keys.

### 3.1. Sample `litellm_config.yaml`

This configuration file defines the model list for LiteLLM, pointing to the Azure ML Online Endpoints.

```yaml
# litellm_config.yaml
model_list:
  - model_name: pacer-mistral-7b
    litellm_params:
      model: azure/pacer-mistral-7b-ep # This is the endpoint name in Azure ML
      api_key: os.environ/PACER_MISTRAL_API_KEY # Securely loaded from environment variable
      api_base: os.environ/PACER_MISTRAL_API_BASE # Scoring URI of the Azure ML endpoint
      # Optional: Add deployment_name if you have multiple deployments under one endpoint
      # deployment_name: "blue" 
  - model_name: builder-llama-3-8b
    litellm_params:
      model: azure/builder-llama-3-8b-ep
      api_key: os.environ/BUILDER_LLAMA3_API_KEY
      api_base: os.environ/BUILDER_LLAMA3_API_BASE
  - model_name: maven-phi-3-mini
    litellm_params:
      model: azure/maven-phi-3-mini-ep
      api_key: os.environ/MAVEN_PHI3_API_KEY
      api_base: os.environ/MAVEN_PHI3_API_BASE
  - model_name: guardian-codellama-7b
    litellm_params:
      model: azure/guardian-codellama-7b-ep
      api_key: os.environ/GUARDIAN_CODELLAMA_API_KEY
      api_base: os.environ/GUARDIAN_CODELLAMA_API_BASE
  - model_name: artisan-codegemma-7b
    litellm_params:
      model: azure/artisan-codegemma-7b-ep
      api_key: os.environ/ARTISAN_CODEGEMMA_API_KEY
      api_base: os.environ/ARTISAN_CODEGEMMA_API_BASE

litellm_settings:
  drop_params: True # Good practice to drop azure specific params before sending to model
  set_verbose: True # For debugging, can be turned to False in production
  # Optional: Add other LiteLLM settings like caching, fallbacks, etc.

# Environment variable mapping for API keys and bases will be:
# PACER_MISTRAL_API_KEY="your_azure_ml_endpoint_key_for_pacer"
# PACER_MISTRAL_API_BASE="https_pacer-mistral-7b-ep.eastus2.inference.ml.azure.com/score" (Example URI)
# ... and so on for each model.
```

**Security Note on API Keys:**
*   The `os.environ/VAR_NAME` syntax in LiteLLM tells it to load the value from an environment variable.
*   These environment variables in Azure Container Apps should be configured as "Secrets" and can be backed by Azure Key Vault. This means the actual key values are stored in Key Vault, and the Container App references them securely.

### 3.2. Sample `Dockerfile` for LiteLLM

```dockerfile
# Dockerfile for LiteLLM
FROM python:3.10-slim

WORKDIR /app

# Install LiteLLM and any specific model provider SDKs if necessary (though for Azure ML, http requests are standard)
# uvicorn for serving, gunicorn for production worker management
RUN pip install litellm uvicorn gunicorn

# Copy the LiteLLM configuration file into the image
COPY litellm_config.yaml .

# Expose the port LiteLLM will run on (default is 8000)
EXPOSE 8000

# Command to run LiteLLM
# The config file will be loaded by LiteLLM automatically if named config.yaml or specified with --config
# Using gunicorn for a more robust server in production
CMD ["gunicorn", "-k", "uvicorn.workers.UvicornWorker", "-w", "4", "-b", "0.0.0.0:8000", "litellm.main:app"]

# Alternative simpler CMD for development (without gunicorn):
# CMD ["litellm", "--config", "litellm_config.yaml", "--port", "8000"]
```

### 3.3. Deploying LiteLLM as an Azure Container App

1.  **Prerequisites:**
    *   Azure Container Registry (ACR) for storing the Docker image (can be the one associated with AI Studio or a separate one in `rg-app-hosting`).
    *   The `litellm_config.yaml` and `Dockerfile` created as above.
    *   Azure CLI or Azure Portal access.

2.  **Build and Push Docker Image:**
    *   `az login`
    *   `az acr login --name <your_acr_name>`
    *   `docker build -t <your_acr_name>.azurecr.io/litellm-proxy:latest .`
    *   `docker push <your_acr_name>.azurecr.io/litellm-proxy:latest`

3.  **Create Azure Key Vault (if not already available from Phase 1):**
    *   Store the API keys for your Azure ML Online Endpoints as secrets in Key Vault (e.g., `PACER-MISTRAL-API-KEY`, `BUILDER-LLAMA3-API-KEY`, etc.).
    *   Store the API base URIs as secrets or directly in environment variables of the Container App if they are not considered highly sensitive (though Key Vault is better practice for consistency).

4.  **Deploy Azure Container App:**
    *   Navigate to your `rg-app-hosting-<env>` resource group.
    *   Create a new "Container App".
    *   **Container App Environment:** Select the CAE created in Phase 1 (`cae-<env>-<region_shortcode>`), which is VNet integrated.
    *   **Basics:**
        *   Container App name: e.g., `litellm-proxy-app`
        *   Region: Same as your CAE.
    *   **App Settings:**
        *   Image Source: `Azure Container Registry`
        *   Registry: Select your ACR.
        *   Image: `litellm-proxy`
        *   Tag: `latest`
        *   **Environment Variables:**
            *   For each model's API key and base URI:
                *   Name: e.g., `PACER_MISTRAL_API_KEY`, `PACER_MISTRAL_API_BASE`.
                *   Source: Click "Reference secret in Key Vault".
                *   Select your Key Vault, the secret name, and version.
            *   This ensures your keys are not exposed directly in the Container App configuration but are fetched at runtime.
            *   You can also set `LITELLM_CONFIG_PATH: /app/litellm_config.yaml` if it's not loading by default, though LiteLLM usually finds `config.yaml` in the workdir.
            *   Set `PORT: 8000` (or whatever port you configured in the Dockerfile CMD).
    *   **Ingress:**
        *   Enable Ingress.
        *   Traffic: `Accepting traffic from anywhere` (if public internet access is desired for initial testing) OR `Limited to Container Apps Environment` (for internal access only, which is recommended for production).
        *   Target Port: `8000` (or the port your container exposes).
        *   If limiting to Container Apps Environment, the LiteLLM proxy will only be accessible from other apps within the same CAE or from services within the VNet that can reach the CAE's internal IP addresses.
    *   **Scaling:**
        *   Configure scale rules similar to other Container Apps.
        *   Minimum replicas: `0` (for scale-to-zero if acceptable for your use case, saves cost but introduces cold start).
        *   Maximum replicas: Start with `1` or `2` and adjust based on load.
        *   Add HTTP scaling rules based on concurrent requests per replica (e.g., scale up if requests > 50 per replica).
    *   **Networking:**
        *   Ensure it's part of the VNet-integrated Container Apps Environment.
        *   If Ingress is internal, note the internal IP for access from other VNet resources.

5.  **Assign Managed Identity (for Key Vault Access):**
    *   Go to the "Identity" section of your created Container App.
    *   Enable System-assigned Managed Identity.
    *   Go to your Azure Key Vault -> Access Policies (or RBAC if using RBAC mode).
    *   Add an access policy (or RBAC role assignment like "Key Vault Secrets User") for the Container App's managed identity, granting it "Get" permissions for secrets.

6.  **Review and Create.**

7.  **Verify Deployment:**
    *   Check logs for any errors.
    *   If ingress is internal, test from another service within the VNet (e.g., a jump box VM or another container app).
    *   If ingress is external, use the provided FQDN to send test requests to LiteLLM. Example: `curl -X POST <litellm_app_fqdn>/chat/completions -H "Content-Type: application/json" -d '{ "model": "pacer-mistral-7b", "messages": [{"role": "user", "content":"Hello, how are you?"}] }'`

This setup provides a secure and scalable way to manage and interact with your AI Council LLMs.
The API keys for the downstream Azure ML endpoints are securely stored in Key Vault and accessed by LiteLLM via its managed identity. LiteLLM then acts as the central, authenticated entry point for your applications to consume these LLMs.

---

# Phase 3: The TDD Engine - AI-Powered Test-Driven Development

This phase details the setup of an automated Test-Driven Development (TDD) pipeline using GitHub Actions, orchestrated by the "Pacer" LLM and leveraging the other AI Council members for specific tasks. The pipeline will take a feature description, generate tests, generate code to pass those tests, run the tests, and iterate if necessary.

**Disclaimer:** The `tdd_pipeline.yml` provided below uses **mocked AI interactions and file manipulations**. Actual implementation would require:
*   Scripts to make HTTP POST requests to the LiteLLM endpoint for each AI Council member.
*   Robust parsing of LLM responses (e.g., JSON, YAML).
*   Intelligent file creation and modification based on LLM outputs.
*   Actual test execution commands for each language.
*   Logic for handling LLM errors, retries, and potentially human feedback loops.

## 1. Conceptual Repository Setup (Polyglot Project)

A clear repository structure is crucial for managing a polyglot project and for the AI agents to understand where to place files.

```
.
├── .github/
│   └── workflows/
│       └── tdd_pipeline.yml  # Our main CI/CD pipeline
├── src/
│   ├── python/
│   │   ├── app/              # Python application code
│   │   │   └── __init__.py
│   │   └── tests/            # Python tests
│   │       └── __init__.py
│   ├── nodejs/
│   │   ├── services/         # Node.js service code
│   │   │   └── service.js
│   │   └── tests/            # Node.js tests
│   │       └── service.test.js
│   └── java/
│       ├── src/main/java/    # Java source code
│       │   └── com/example/
│       │       └── Main.java
│       └── src/test/java/    # Java tests
│           └── com/example/
│               └── MainTest.java
├── docs/                     # Project documentation
├── scripts/                  # Helper scripts (e.g., for calling LLMs, parsing)
│   └── call_llm.sh           # Example script placeholder
└── README.md
```

This structure separates source code and tests by language, making it easier for both humans and AI to navigate and manage.

## 2. GitHub Actions Workflow: `tdd_pipeline.yml`

This workflow automates the TDD process.

```yaml
# .github/workflows/tdd_pipeline.yml
name: AI TDD Engine

on:
  workflow_dispatch:
    inputs:
      feature_description:
        description: 'Detailed description of the new feature or improvement'
        required: true
        type: string
      target_languages:
        description: 'Comma-separated list of languages to implement (e.g., python,nodejs,java)'
        required: true
        type: string
        default: 'python'
      pr_branch_name:
        description: 'Name for the new branch to create the PR from (e.g., feature/new-widget)'
        required: true
        type: string
  pull_request:
    branches:
      - main # Or your development branch
    # Optional: Trigger on specific labels or comments in a PR for refinement loops

env:
  LITELLM_ENDPOINT_URL: ${{ secrets.LITELLM_API_ENDPOINT }} # URL for your LiteLLM Azure Container App
  LITELLM_API_KEY: ${{ secrets.LITELLM_API_KEY }} # If LiteLLM itself is protected by a key

jobs:
  generate_tests:
    runs-on: ubuntu-latest
    outputs:
      test_plan_id: ${{ steps.pacer_call.outputs.plan_id }}
      languages: ${{ github.event.inputs.target_languages || 'python' }} # Default if not from dispatch
      pr_branch: ${{ github.event.inputs.pr_branch_name || format('ai-tdd/{0}-{1}', github.run_id, github.run_attempt) }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref || github.ref }} # Checkout PR branch or current ref

      - name: Create new branch for AI work (if workflow_dispatch)
        if: github.event_name == 'workflow_dispatch'
        run: |
          git checkout -b ${{ github.event.inputs.pr_branch_name }}
          git push --set-upstream origin ${{ github.event.inputs.pr_branch_name }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: "Debug: Initial branch info"
        run: |
          echo "Running on branch: $(git rev-parse --abbrev-ref HEAD)"
          echo "Target languages: ${{ github.event.inputs.target_languages || 'python' }}"
          echo "PR Branch Name: ${{ github.event.inputs.pr_branch_name || 'N/A' }}"

      - name: "[MOCK] Call Pacer (LLM) for Test Plan"
        id: pacer_call
        run: |
          echo "Feature Description: ${{ github.event.inputs.feature_description }}"
          echo "Target Languages: ${{ github.event.inputs.target_languages }}"
          echo "Simulating Pacer LLM: Generating test plan..."
          # Real implementation: call_llm.sh pacer-mistral-7b --prompt "Create a test plan for feature: ${{ github.event.inputs.feature_description }} for languages ${{ github.event.inputs.target_languages }}"
          mkdir -p .ai_artifacts/pacer
          echo '{"plan_id": "plan_123", "tests_to_generate": [{"language": "python", "description": "Test for user login"}, {"language": "nodejs", "description": "Test for API endpoint"}]}' > .ai_artifacts/pacer/test_plan.json
          echo "plan_id=plan_$(date +%s)" >> $GITHUB_OUTPUT # Unique ID for this plan
          echo "Test plan generated: .ai_artifacts/pacer/test_plan.json"

      - name: "[MOCK] Call Maven (LLM) for Context/RAG"
        id: maven_call
        run: |
          echo "Simulating Maven LLM: Retrieving relevant context, code snippets, docs for plan_id ${{ steps.pacer_call.outputs.plan_id }}..."
          # Real implementation: call_llm.sh maven-phi-3-mini --prompt "Gather context for test plan ${{ steps.pacer_call.outputs.plan_id }}" --input_files "src/**"
          mkdir -p .ai_artifacts/maven
          echo "Relevant context for plan_123..." > .ai_artifacts/maven/context.txt
          echo "Context gathered: .ai_artifacts/maven/context.txt"

      - name: "[MOCK] Call Builder (LLM) to Generate Tests"
        id: builder_generate_tests
        run: |
          echo "Simulating Builder LLM: Generating test files based on plan and context..."
          # Real implementation: Loop through test_plan.json, call Builder for each test
          # call_llm.sh builder-llama-3-8b --prompt "Generate test code for: Test for user login (Python)" --context_file .ai_artifacts/maven/context.txt
          mkdir -p .ai_artifacts/builder/tests/python/tests .ai_artifacts/builder/tests/nodejs/tests
          echo "def test_user_login(): assert True" > .ai_artifacts/builder/tests/python/tests/test_login.py
          echo "console.log('Mock Node.js test for API endpoint');" > .ai_artifacts/builder/tests/nodejs/tests/test_api.js
          echo "Test files generated in .ai_artifacts/builder/tests/"
          # In a real scenario, these files would be placed in the correct src/LANG/tests/ path and committed.
          # For this mock, we use artifacts.

      - name: Upload Test Plan and Generated Tests
        uses: actions/upload-artifact@v4
        with:
          name: ai-generated-tests-${{ steps.pacer_call.outputs.plan_id }}
          path: |
            .ai_artifacts/pacer/test_plan.json
            .ai_artifacts/maven/context.txt
            .ai_artifacts/builder/tests/

  generate_code:
    needs: generate_tests
    runs-on: ubuntu-latest
    outputs:
      pr_branch: ${{ needs.generate_tests.outputs.pr_branch }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ needs.generate_tests.outputs.pr_branch }} # Checkout the branch created/used in generate_tests

      - name: Download Test Plan and Generated Tests
        uses: actions/download-artifact@v4
        with:
          name: ai-generated-tests-${{ needs.generate_tests.outputs.test_plan_id }}
          path: .ai_artifacts/

      - name: "[MOCK] Call Builder (LLM) to Generate Code"
        id: builder_generate_code
        run: |
          echo "Simulating Builder LLM: Generating source code to pass the tests..."
          # Real implementation: Loop through tests, call Builder to generate implementation code
          # e.g., call_llm.sh builder-llama-3-8b --prompt "Generate Python code to pass test_login.py" --test_file .ai_artifacts/builder/tests/python/tests/test_login.py --context_file .ai_artifacts/context.txt
          mkdir -p .ai_artifacts/builder/src/python/app .ai_artifacts/builder/src/nodejs/services
          echo "def user_login_feature(): return True" > .ai_artifacts/builder/src/python/app/login.py
          echo "function apiEndpointFeature() { console.log('API logic'); return true; }" > .ai_artifacts/builder/src/nodejs/services/api.js
          echo "Source code generated in .ai_artifacts/builder/src/"
          # Real: Place files in actual src/LANG/app_or_service path and commit.

      - name: Upload Generated Source Code
        uses: actions/upload-artifact@v4
        with:
          name: ai-generated-src-${{ needs.generate_tests.outputs.test_plan_id }}
          path: .ai_artifacts/builder/src/

  run_tests:
    needs: [generate_tests, generate_code]
    runs-on: ubuntu-latest
    outputs:
      test_outcome: ${{ steps.determine_outcome.outputs.outcome }} # success or failure
      pr_branch: ${{ needs.generate_code.outputs.pr_branch }}
    strategy:
      matrix:
        language: ${{ fromJson(format('["%s"]', replace(needs.generate_tests.outputs.languages, ',', '","'))) }} # Dynamically create matrix from input
      fail-fast: false # Allow all language tests to run even if one fails
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ needs.generate_code.outputs.pr_branch }}

      - name: Download Generated Tests
        uses: actions/download-artifact@v4
        with:
          name: ai-generated-tests-${{ needs.generate_tests.outputs.test_plan_id }}
          path: .ai_artifacts-tests/ # Separate path to avoid conflicts if src is also downloaded

      - name: Download Generated Source Code
        uses: actions/download-artifact@v4
        with:
          name: ai-generated-src-${{ needs.generate_tests.outputs.test_plan_id }}
          path: .ai_artifacts-src/

      - name: "[MOCK] Setup Environment for ${{ matrix.language }}"
        run: |
          echo "Setting up environment for ${{ matrix.language }}..."
          # Real: Install Python, Node.js, JDK, Maven/Gradle, dependencies etc.
          # if [ "${{ matrix.language }}" == "python" ]; then pip install -r src/python/requirements.txt; fi
          # if [ "${{ matrix.language }}" == "nodejs" ]; then cd src/nodejs && npm install; cd ../..; fi

      - name: "[MOCK] Copy generated files to workspace for ${{ matrix.language }}"
        # This step simulates committing the files to the branch.
        # In a real workflow, generate_tests and generate_code would commit their files.
        run: |
          echo "Copying generated files for ${{ matrix.language }}..."
          # Example for Python:
          if [ "${{ matrix.language }}" == "python" ]; then
            mkdir -p src/python/tests src/python/app
            cp -r .ai_artifacts-tests/builder/tests/python/tests/* src/python/tests/ || echo "No Python tests found in artifact"
            cp -r .ai_artifacts-src/builder/src/python/app/* src/python/app/ || echo "No Python src found in artifact"
          fi
          # Example for Node.js:
          if [ "${{ matrix.language }}" == "nodejs" ]; then
            mkdir -p src/nodejs/tests src/nodejs/services
            cp -r .ai_artifacts-tests/builder/tests/nodejs/tests/* src/nodejs/tests/ || echo "No Node.js tests found in artifact"
            cp -r .ai_artifacts-src/builder/src/nodejs/services/* src/nodejs/services/ || echo "No Node.js src found in artifact"
          fi
          # Add similar blocks for Java, etc.
          echo "Listing files:"
          ls -R src/

      - name: "[MOCK] Run ${{ matrix.language }} Tests"
        run: |
          echo "Running ${{ matrix.language }} tests..."
          # Real implementation:
          # if [ "${{ matrix.language }}" == "python" ]; then pytest src/python/tests/; fi
          # if [ "${{ matrix.language }}" == "nodejs" ]; then cd src/nodejs && npm test; cd ../..; fi
          # if [ "${{ matrix.language }}" == "java" ]; then mvn -f src/java/pom.xml test; fi
          if [ "${{ matrix.language }}" == "python" ] && [ ! -f src/python/tests/test_login.py ]; then echo "Python test file missing!"; exit 1; fi
          if [ "${{ matrix.language }}" == "nodejs" ] && [ ! -f src/nodejs/tests/test_api.js ]; then echo "Node.js test file missing!"; exit 1; fi
          # Simulate test failure for demonstration occasionally
          # if (( RANDOM % 2 == 0 )); then echo "Mock test failure!"; exit 1; fi

      - name: "[MOCK] Determine Test Outcome"
        id: determine_outcome
        # This step would aggregate results if matrix strategy was used for individual test files
        # For now, it assumes success if the previous step didn't exit with error
        run: |
          echo "Tests for ${{ matrix.language }} completed."
          echo "outcome=success" >> $GITHUB_OUTPUT # Default to success for mock
          # In reality, parse test reports to determine success/failure.

  refine_or_review:
    needs: [generate_tests, run_tests] # Depends on generate_tests for branch name
    runs-on: ubuntu-latest
    # Only run if tests failed OR if all tests passed (for review)
    # This logic is complex for matrix outcomes; simplified here.
    # A more robust solution might involve a separate job to aggregate matrix 'run_tests' outcomes.
    # For this mock, we'll assume 'run_tests' provides an overall outcome, or just run if any test job failed.
    # if: needs.run_tests.outputs.test_outcome == 'failure' || always() # 'always()' ensures it runs for review if tests pass
    if: always() # Simplified: always run this job to decide next steps.
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ needs.run_tests.outputs.pr_branch }} # Use the branch from run_tests

      - name: Download All Artifacts (if needed for context)
        uses: actions/download-artifact@v4
        with:
          name: ai-generated-tests-${{ needs.generate_tests.outputs.test_plan_id }}
          path: .ai_artifacts/tests
        continue-on-error: true
      - uses: actions/download-artifact@v4
        with:
          name: ai-generated-src-${{ needs.generate_tests.outputs.test_plan_id }}
          path: .ai_artifacts/src
        continue-on-error: true

      - name: "[MOCK] Decide Next Step based on Test Outcomes"
        id: decision
        run: |
          # In a real scenario, you'd check the actual outcomes of all matrix jobs in 'run_tests'.
          # This requires more complex scripting to fetch job conclusions via GitHub API or by passing structured outputs.
          # For this MOCK, we'll simulate a failure sometimes to show the refine path.
          # This is a placeholder for more robust outcome aggregation.
          # Assume 'needs.run_tests.outputs.test_outcome' gives an overall status.
          # However, 'needs.run_tests.outputs' isn't directly accessible due to matrix.
          # Let's simulate:
          if (( RANDOM % 3 == 0 )); then # Roughly 33% chance of simulated failure
            echo "Simulated test failure detected from 'run_tests' job."
            echo "action=refine" >> $GITHUB_OUTPUT
          else
            echo "Simulated test success from 'run_tests' job."
            echo "action=review" >> $GITHUB_OUTPUT
          fi

      - name: "[MOCK] Call Artisan (LLM) to Refine Code (if tests failed)"
        if: steps.decision.outputs.action == 'refine'
        run: |
          echo "Simulating Artisan LLM: Refining code based on test failures..."
          # Real: call_llm.sh artisan-codegemma-7b --prompt "Refine code in [file] due to test failure: [error]" --code_file ... --test_output ...
          echo "Refined code would be generated and committed here."
          # This might loop back to 'run_tests' or require human intervention.
          # For now, we'll just log it.
          echo "Code refinement requested. In a full loop, would re-run tests or create PR for review."

      - name: "[MOCK] Call Guardian (LLM) for Code Review (if tests passed)"
        if: steps.decision.outputs.action == 'review'
        run: |
          echo "Simulating Guardian LLM: Performing code review and security check..."
          # Real: call_llm.sh guardian-codellama-7b --prompt "Review code for quality, security, and best practices" --code_files "src/**"
          mkdir -p .ai_artifacts/guardian
          echo "Guardian Review: Looks good, but consider edge cases for login. No major security issues found." > .ai_artifacts/guardian/review_comments.txt
          echo "Review comments generated: .ai_artifacts/guardian/review_comments.txt"

      - name: "[MOCK] Commit files and Create/Update Pull Request"
        # This step would normally commit any new/modified files from LLMs (tests, src, review comments)
        # and then create or update a Pull Request.
        run: |
          echo "Branch for PR: ${{ needs.run_tests.outputs.pr_branch }}"
          # Mock creating/updating files
          git config --global user.name "AI TDD Bot"
          git config --global user.email "ai-tdd-bot@example.com"
          
          # Simulate adding review comments to a file in the PR
          if [ -f ".ai_artifacts/guardian/review_comments.txt" ]; then
            mkdir -p CODE_REVIEW
            cp .ai_artifacts/guardian/review_comments.txt CODE_REVIEW/guardian_review.md
            git add CODE_REVIEW/guardian_review.md || echo "No review file to add"
          fi

          # Simulate adding generated code/tests (in a real scenario, this happens in earlier jobs)
          # This is simplified; actual file paths from artifacts would be used.
          if [ -d "src" ]; then # If 'src' was populated by 'run_tests' mock copy
             git add src/
          fi

          git commit -m "AI TDD Cycle: ${{ github.event.inputs.feature_description }} (run ${{ github.run_id }})" || echo "No changes to commit."
          git push origin ${{ needs.run_tests.outputs.pr_branch }} --force || echo "Failed to push changes or no changes to push."
          
          echo "If this were a real run for workflow_dispatch, a PR would be created/updated now."
          echo "Example: gh pr create --title 'AI TDD: ${{ github.event.inputs.feature_description }}' --body 'Automated PR by AI TDD Engine. Review comments in CODE_REVIEW/guardian_review.md' --base main --head ${{ needs.run_tests.outputs.pr_branch }}"
          # For PR triggered runs, it would add comments or push new commits.
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # GITHUB_TOKEN has permissions to create PRs in the same repo
          # For cross-repo PRs or more complex scenarios, a PAT might be needed.
```

## 3. Explanations for Key Aspects

*   **Secret Management:**
    *   `LITELLM_API_ENDPOINT` and `LITELLM_API_KEY` (if LiteLLM is secured) are stored as GitHub encrypted secrets.
    *   `GITHUB_TOKEN` is automatically available and used for actions like checking out code and potentially creating PRs (within the same repository).

*   **Prompt Engineering:**
    *   **Pacer:** Needs clear feature descriptions and target languages to generate a comprehensive test plan. Prompts should ask for structured output (e.g., JSON list of test cases with language and description).
    *   **Maven:** Needs the test plan (or specific parts) to fetch relevant context. Prompts should guide it to look for existing code, documentation, or patterns.
    *   **Builder (Tests):** Needs a specific test case description from Pacer's plan and context from Maven. Prompts should specify the testing framework (if known) and desired level of detail.
    *   **Builder (Code):** Needs the generated test file(s) and context. Prompts should instruct it to write code that makes the tests pass, adhering to project conventions.
    *   **Artisan:** Needs the failing test output and the relevant code. Prompts should focus on fixing the error or optimizing the existing code.
    *   **Guardian:** Needs the generated code and tests. Prompts should ask for review based on coding standards, security vulnerabilities, and best practices. Output should be structured for easy parsing (e.g., list of comments with severity and file/line numbers).
    *   **General:** Use few-shot prompting (providing examples) if initial results are poor. Clearly define roles and expected output formats for each LLM.

*   **LLM Output Parsing:**
    *   LLMs should be prompted to return structured data (JSON, YAML).
    *   The scripts (e.g., `call_llm.sh` placeholder) would need to parse this output. For example, if Pacer returns a JSON array of tests, the workflow needs to iterate over it.
    *   Robust error handling is needed for malformed LLM responses.

*   **State Management Between AI Calls:**
    *   **GitHub Artifacts:** Used in the mock to pass generated files (test plans, context, test code, source code, review comments) between jobs.
    *   **Committing to Branch:** In a real implementation, each job that generates or modifies files (tests, source code) would commit those changes to the PR branch. This makes the state persistent and visible. The subsequent jobs would pull the latest changes from the branch.
    *   **Files with IDs:** Using unique IDs (e.g., `plan_id`) can help correlate artifacts and logs across different stages.

*   **Polyglot Test Execution:**
    *   The `run_tests` job uses a `matrix` strategy based on the `target_languages` input.
    *   Each matrix instance would:
        1.  Set up the specific environment for that language (e.g., `actions/setup-python`, `actions/setup-node`, `actions/setup-java`).
        2.  Install dependencies (`pip install`, `npm install`, `mvn dependency:resolve`).
        3.  Run the tests using the language-specific test runner (`pytest`, `npm test`, `mvn test`).
        4.  Parse test results to determine success/failure.

*   **Error Handling & Iteration:**
    *   `continue-on-error: true` can be used on steps if partial failure is acceptable or handled by later steps.
    *   Job dependencies (`needs: [...]`) control the execution flow.
    *   The `refine_or_review` job demonstrates a conditional step. If tests fail, Artisan is called.
    *   A true iterative loop (e.g., Code -> Test -> Refine -> Test) would be more complex, potentially involving:
        *   Re-running the `run_tests` job after Artisan makes changes.
        *   Using workflow dispatch or specific PR comments/labels to trigger refinement loops.
        *   Setting a maximum number of retries to prevent infinite loops.

*   **Importance of Human Review:**
    *   **Crucial Final Step:** AI-generated code and tests **must** be reviewed by human developers before merging.
    *   The pipeline aims to automate the TDD boilerplate and provide a strong starting point, not replace human oversight.
    *   Guardian's output can feed into the PR description or as comments to aid human reviewers.
    *   The final step should be a clear PR ready for human assessment.

*   **Managing GitHub Actions Costs:**
    *   **Self-Hosted Runners:** For GPU-intensive LLM calls (if running models locally, not via API) or long-running jobs, self-hosted runners can be more cost-effective than GitHub-hosted runners. However, this pipeline calls external LLM APIs, so compute is on Azure.
    *   **Concurrency Limits:** Be mindful of concurrency limits on GitHub-hosted runners for your account tier.
    *   **Optimize Workflows:**
        *   Ensure jobs only run when necessary.
        *   Use caching for dependencies (`actions/cache`).
        *   Mocking (as done here) is cheap; actual LLM API calls will incur costs on Azure based on token usage per model.
        *   Implement scale-to-zero for your Azure ML Endpoints and LiteLLM Container App to save costs when idle.
    *   **Artifact Storage:** Artifacts also consume storage, which has associated costs. Regularly clean up old artifacts if space becomes an issue.

*   **Mocked Interactions Clarification:**
    *   The YAML workflow uses `echo` statements and basic file creation (e.g., `echo "def test_user_login(): assert True" > ...`) to simulate what real scripts interacting with LLMs would do.
    *   **Actual Implementation:** Each "[MOCK] Call..." step would be replaced by a script execution (e.g., `bash ./scripts/call_llm.sh <model_name> --prompt "..." --output_file ...`) that handles:
        1.  Constructing the correct JSON payload for the LiteLLM endpoint.
        2.  Making an HTTP POST request to the `LITELLM_ENDPOINT_URL`.
        3.  Handling the response (e.g., extracting the LLM's message).
        4.  Parsing the LLM's message (e.g., if it's JSON or contains code).
        5.  Writing the relevant content to files in the workspace (e.g., test files, source code files).
        6.  Committing these new/modified files to the current branch so subsequent jobs can use them.

This TDD engine provides a powerful framework for AI-assisted development. The key is iterative refinement of prompts, robust parsing, and always keeping a human in the loop for final validation.

---

# Phase 4: Polyglot Code Generation Demonstration - User Profile Display

This phase demonstrates the AI Council's capability in handling a polyglot feature request, specifically a "User Profile Display." We'll show example prompts for generating tests (Guardian), code (Builder, guided by Pacer's plan), and refining code (Artisan) for both Python/FastAPI backend and TypeScript/React frontend components. We'll also discuss how Guardian handles polyglot test generation and how linters/formatters are managed.

## 1. Example Feature: User Profile Display

**Feature Request:**
"As a user, I want to view my user profile information, including my username, email, full name, and join date. The backend should expose an API endpoint to fetch this data, and the frontend should display it clearly on a profile page."

**Technical Stack:**
*   **Backend:** Python 3.9+, FastAPI, Pytest
*   **Frontend:** TypeScript, React, Jest, React Testing Library

## 2. Example Prompts for the AI Council

These prompts are conceptual and would be sent to the respective LLMs via the LiteLLM proxy. They include bracketed placeholders like `[User Story]` or `[Test File Content]` which would be dynamically filled by the orchestrating GitHub Actions workflow.

### 2.1. Guardian: Generating Tests

Guardian is prompted first to generate tests based on Pacer's breakdown of the feature. It needs separate, targeted prompts for each language/framework.

**Prompt for Python/Pytest (Backend API Tests):**

```
Role: Guardian (Test Generation Specialist)
Task: Generate Pytest tests for a FastAPI backend endpoint.
Feature: [User Story: "As a user, I want to view my user profile information, including my username, email, full name, and join date. The backend should expose an API endpoint to fetch this data..."]

Details:
1.  The API endpoint should be `/api/v1/users/me/profile`.
2.  It should be a GET request.
3.  Assume the user is authenticated (mock authentication if necessary).
4.  The expected successful response (200 OK) should be a JSON object containing:
    *   `username`: string
    *   `email`: string (valid email format)
    *   `full_name`: string
    *   `join_date`: string (ISO 8601 date format, e.g., "YYYY-MM-DD")
5.  Generate tests for:
    *   Successful retrieval of a user profile.
    *   User not authenticated (expect 401/403 error, mock this).
    *   User profile not found (expect 404 error, if applicable for your data model).
    *   Validate data types and presence of all fields in a successful response.
6.  Use standard Pytest conventions. Mock any database interactions or external service calls.
7.  Output the test code in a single Python file named `test_user_profile_api.py`.
8.  Ensure tests are self-contained and runnable. Include necessary imports (e.g., `FastAPI`, `TestClient`, `pytest`).

Example (minimal) structure for a test:
```python
from fastapi.testclient import TestClient
from main_app import app # Assuming your FastAPI app instance is in main_app.py

client = TestClient(app)

def test_read_user_profile_success():
    # Mock authentication if needed
    response = client.get("/api/v1/users/me/profile" #, headers={"Authorization": "Bearer fake-token"}
    )
    assert response.status_code == 200
    data = response.json()
    assert "username" in data
    assert "email" in data
    # ... more assertions
```
```

**Prompt for TypeScript/Jest (Frontend Component Tests):**

```
Role: Guardian (Test Generation Specialist)
Task: Generate Jest and React Testing Library tests for a React frontend component.
Feature: [User Story: "...the frontend should display it clearly on a profile page."]

Details:
1.  The component will be named `UserProfileDisplay.tsx`.
2.  It will receive user profile data as props:
    ```typescript
    interface UserProfileData {
      username: string;
      email: string;
      fullName: string;
      joinDate: string; // Display as human-readable, e.g., "January 1, 2024"
    }
    ```
3.  Generate tests to:
    *   Render the username, email, full name, and formatted join date correctly from props.
    *   Display a loading state (e.g., "Loading profile...") when `isLoading` prop is true.
    *   Display an error message (e.g., "Failed to load profile.") when `error` prop is provided.
    *   Ensure accessibility (e.g., appropriate roles, aria-labels if complex).
4.  Use Jest and React Testing Library (`@testing-library/react`).
5.  Output the test code in a single TypeScript file named `UserProfileDisplay.test.tsx`.
6.  Ensure tests are self-contained. Import the component to be tested.

Example (minimal) structure for a test:
```typescript
import { render, screen } from '@testing-library/react';
import UserProfileDisplay, { UserProfileData } from './UserProfileDisplay'; // Assuming component path

describe('UserProfileDisplay', () => {
  const mockProfile: UserProfileData = {
    username: 'testuser',
    email: 'test@example.com',
    fullName: 'Test User',
    joinDate: '2024-01-15T10:00:00Z', // Will need formatting
  };

  it('renders user profile information correctly', () => {
    render(<UserProfileDisplay profile={mockProfile} isLoading={false} error={null} />);
    expect(screen.getByText(mockProfile.username)).toBeInTheDocument();
    expect(screen.getByText(mockProfile.email)).toBeInTheDocument();
    // ... more assertions for fullName and formatted joinDate
  });
});
```
```

### 2.2. Pacer (Planning) / Builder (Code Generation)

Pacer would break down the feature into backend and frontend tasks. Builder then receives prompts to generate code to pass the tests created by Guardian.

**Prompt for Python/FastAPI (Backend Code Generation):**

```
Role: Builder (Code Generation Specialist)
Task: Generate Python FastAPI code to implement the user profile API endpoint.
Feature: [User Story: Backend portion for User Profile Display]
Associated Tests: [Content of `test_user_profile_api.py` generated by Guardian]

Details:
1.  Implement the API endpoint `/api/v1/users/me/profile` using FastAPI.
2.  The endpoint should handle GET requests.
3.  It must pass all tests in the provided Pytest file (`test_user_profile_api.py`).
4.  Define a Pydantic model for the response with fields: `username`, `email`, `full_name`, `join_date`.
5.  For now, you can return mock/hardcoded data that satisfies the tests. Do not implement actual database logic or authentication checks; focus on the API structure and test passability.
    *   Mock user data: `{"username": "testuser", "email": "test@example.com", "full_name": "Test User", "join_date": "2023-05-15"}`
6.  Structure the code clearly, potentially in a dedicated router file (e.g., `routers/user_profile.py`) and a models file (e.g., `models/user.py`).
7.  Output the Python code files. Name the main FastAPI application file `main_app.py` if creating a new one, or specify how to integrate into an existing structure.

Example Response Model (in `models/user.py`):
```python
from pydantic import BaseModel, EmailStr
from datetime import date

class UserProfileResponse(BaseModel):
    username: str
    email: EmailStr
    full_name: str
    join_date: date # FastAPI will serialize this to ISO 8601 string
```
```

**Prompt for TypeScript/React (Frontend Code Generation):**

```
Role: Builder (Code Generation Specialist)
Task: Generate TypeScript React code for the `UserProfileDisplay` component.
Feature: [User Story: Frontend portion for User Profile Display]
Associated Tests: [Content of `UserProfileDisplay.test.tsx` generated by Guardian]

Details:
1.  Implement the `UserProfileDisplay.tsx` React component.
2.  The component must pass all tests in the provided Jest/RTL test file (`UserProfileDisplay.test.tsx`).
3.  It should accept props: `profile: UserProfileData | null`, `isLoading: boolean`, `error: string | null`.
    ```typescript
    interface UserProfileData {
      username: string;
      email: string;
      fullName: string;
      joinDate: string; // ISO 8601 string from API
    }
    ```
4.  Display the profile information. Format the `joinDate` (e.g., from "2024-01-15T10:00:00Z" to "January 15, 2024").
5.  Handle `isLoading` and `error` states as per the tests.
6.  Use functional components and hooks.
7.  Output the TypeScript React component file (`UserProfileDisplay.tsx`).

Example component structure:
```typescript
import React from 'react';

interface UserProfileData { /* ... as above ... */ }

interface UserProfileDisplayProps {
  profile: UserProfileData | null;
  isLoading: boolean;
  error: string | null;
}

const UserProfileDisplay: React.FC<UserProfileDisplayProps> = ({ profile, isLoading, error }) => {
  if (isLoading) return <div>Loading profile...</div>;
  if (error) return <div>{error}</div>;
  if (!profile) return <div>No profile data.</div>;

  const formatDate = (isoDate: string) => {
    return new Date(isoDate).toLocaleDateString('en-US', {
      year: 'numeric', month: 'long', day: 'numeric'
    });
  };

  return (
    // JSX to display username, email, fullName, formatted joinDate
  );
};

export default UserProfileDisplay;
```
```

### 2.3. Artisan: Refining Code

Artisan is called if tests fail, or for a general quality pass even if tests pass.

**Prompt for Artisan (Code Refinement - e.g., if a Python test failed):**

```
Role: Artisan (Code Refinement Specialist)
Task: Refine Python FastAPI code to fix failing tests and improve quality.
Feature: [User Story: Backend portion for User Profile Display]
Code Files: [Content of `routers/user_profile.py`, `models/user.py`]
Test File: [Content of `test_user_profile_api.py`]
Test Output/Error: [Specific error message from Pytest, e.g., "AssertionError: Expected status code 200 but got 404 on /api/v1/users/me/profile"]

Details:
1.  Analyze the provided code and the failing test output.
2.  Identify the root cause of the test failure.
3.  Modify the Python code to make the test pass.
4.  Additionally, review the code for:
    *   Adherence to FastAPI best practices.
    *   Clarity and readability.
    *   Proper error handling (even if not explicitly in the failing test).
    *   Efficient data handling (though current focus is mock data).
5.  Output the refined Python code files. Explain the changes made.
```

**Prompt for Artisan (General Frontend Refinement - even if tests pass):**

```
Role: Artisan (Code Refinement Specialist)
Task: Refine the `UserProfileDisplay.tsx` React component for best practices and robustness.
Feature: [User Story: Frontend portion for User Profile Display]
Code File: [Content of `UserProfileDisplay.tsx`]
Test File: [Content of `UserProfileDisplay.test.tsx`]

Details:
1.  Review the React component for:
    *   React best practices (e.g., hook usage, component composition, memoization if applicable).
    *   TypeScript best practices (e.g., type safety, clarity of interfaces).
    *   Accessibility improvements beyond basic test coverage.
    *   Potential edge cases or UI inconsistencies.
    *   Code style and readability.
    *   Consider internationalization (i18n) needs for date formatting if applicable (though current prompt uses `toLocaleDateString`).
2.  Output the refined `UserProfileDisplay.tsx` file. Explain any significant changes or recommendations.
```

## 3. Guardian's Polyglot Test Generation

Guardian handles polyglot test generation by:

1.  **Targeted Prompts:** As seen above, Guardian receives distinct prompts for each language and testing framework. The prompt includes specific instructions, expected file names, and examples relevant to that language's ecosystem (Pytest for Python, Jest/RTL for TypeScript/React).
2.  **Specialized Knowledge:** Guardian (as an LLM fine-tuned or prompted for code understanding) has knowledge of different programming languages, their common testing frameworks, and idiomatic test structures.
3.  **File Naming Conventions:** The prompts explicitly request specific file names that align with common conventions for each language:
    *   Python: `test_*.py` or `*_test.py` (e.g., `test_user_profile_api.py`).
    *   TypeScript/JavaScript (Jest): `*.test.ts`, `*.spec.ts`, `*.test.tsx`, `*.spec.tsx` (e.g., `UserProfileDisplay.test.tsx`).
    The TDD workflow would then know where to place these files within the repository structure (e.g., `src/python/tests/`, `src/nodejs/tests/`).
4.  **Separate Outputs:** Guardian generates each set of tests as a separate output (e.g., one block of code for Python tests, another for TypeScript tests). The GitHub Actions workflow is responsible for routing these to the correct locations in the workspace or artifacts.

## 4. Managing Linters and Formatters in the TDD Workflow

Integrating linters and formatters is crucial for maintaining code quality, consistency, and readability, especially with AI-generated code.

**General Strategy:**
*   Linters and formatters are run automatically in the GitHub Actions workflow after code generation (and potentially after refinement).
*   They can be configured to `--check` for violations (fail the build) or to automatically fix issues and commit changes. The latter is often preferred for formatters.

**Language-Specific Configurations:**

*   **Python (Black & Flake8/Ruff):**
    *   **Configuration:**
        *   `pyproject.toml`: Configure Black (line length, target Python versions) and Flake8 (or Ruff, which is faster and can replace Flake8, isort).
        ```toml
        # pyproject.toml example
        [tool.black]
        line-length = 88
        target-version = ['py39']

        [tool.flake8] # Or [tool.ruff]
        max-line-length = 88
        extend-ignore = "E203,W503" # Example ignores
        # For Ruff:
        # [tool.ruff]
        # line-length = 88
        # select = ["E", "F", "W", "I"] # Select rules
        ```
    *   **CI Steps (in `tdd_pipeline.yml` under `run_tests` or a dedicated linting job):**
        ```yaml
        - name: Setup Python
          uses: actions/setup-python@v4
          with:
            python-version: '3.9'
        - name: Install Python Linters
          run: pip install black flake8 # or ruff
        - name: Run Black Formatter Check
          run: black --check . --config pyproject.toml
        - name: Run Flake8 Linter # or Ruff
          run: flake8 . --config pyproject.toml # or ruff check . --config pyproject.toml
        # Optional: Auto-fix and commit (more complex, requires PAT or app with write perms)
        # - name: Auto-format with Black (if check fails)
        #   if: failure() # Or a specific condition based on black --check output
        #   run: |
        #     black . --config pyproject.toml
        #     git config --global user.name 'GitHub Actions Linter'
        #     git config --global user.email 'actions@github.com'
        #     git commit -am "Chore: Auto-format Python code with Black" || echo "No changes to commit"
        #     git push
        ```

*   **JavaScript/TypeScript (ESLint & Prettier):**
    *   **Configuration:**
        *   `.eslintrc.js` (or `.json`, `.yaml`): Configure ESLint rules, plugins (e.g., `@typescript-eslint/eslint-plugin`, `eslint-plugin-react`, `eslint-plugin-jest`).
        *   `.prettierrc.js` (or `.json`, `.yaml`): Configure Prettier formatting options (tab width, semi-colons, print width).
        *   `package.json`: Add linting/formatting scripts.
        ```json
        // package.json (scripts section)
        "scripts": {
          "lint": "eslint 'src/**/*.{js,jsx,ts,tsx}' --quiet",
          "format:check": "prettier --check 'src/**/*.{js,jsx,ts,tsx,json,css,md}'",
          "format:write": "prettier --write 'src/**/*.{js,jsx,ts,tsx,json,css,md}'"
        }
        ```
    *   **CI Steps (in `tdd_pipeline.yml` under `run_tests` or a dedicated linting job):**
        ```yaml
        - name: Setup Node.js
          uses: actions/setup-node@v4
          with:
            node-version: '18' # Or your project's version
        - name: Install JS/TS Dependencies (including linters)
          # Assuming linters are in devDependencies
          run: npm ci # or yarn install --frozen-lockfile
        - name: Run ESLint
          run: npm run lint
        - name: Run Prettier Check
          run: npm run format:check
        # Optional: Auto-fix and commit (similar to Python example)
        # - name: Auto-format with Prettier (if check fails)
        #   if: failure()
        #   run: |
        #     npm run format:write
        #     # Git commit and push logic here
        ```

**Workflow Integration:**

1.  **Placement:** Linting/formatting steps should ideally run *before* tests, as syntactic errors or major formatting issues can sometimes prevent tests from running correctly. However, they can also run in parallel or after tests if the primary goal is to enforce standards on code that is already functionally tested.
2.  **Failure Policy:** Decide if linter/formatter violations should fail the build (`--check` mode). For formatters, it's common to have a separate step that auto-fixes and commits, or to provide instructions for developers to fix locally.
3.  **Feedback to LLMs:** If linters/formatters fail on AI-generated code, this feedback could potentially be passed back to Artisan for a refinement iteration, prompting it to adhere to specific style guides. This adds complexity but could improve the quality of direct AI output.
    *   Example prompt addition for Artisan: "Additionally, ensure the generated code passes `flake8` with the provided `pyproject.toml` configuration and `black` formatting."

By incorporating these practices, the TDD engine can ensure that the AI-generated code is not only functional (passes tests) but also clean, consistent, and maintainable across different languages.

---

# Phase 5: Autonomous Operation and Cost Management

This phase outlines strategies for monitoring the AI Council and associated infrastructure, managing costs effectively, and implementing automation for GPU resource scaling to ensure operational efficiency and cost-effectiveness.

## 1. Monitoring Strategy (Azure Monitor)

A comprehensive monitoring strategy is crucial for observing the health, performance, and cost of the deployed AI solution. Azure Monitor will be the central tool for this.

### 1.1. Centralized Logging

*   **Log Analytics Workspace:** All logs from Azure Container Apps (hosting LiteLLM) and Azure AI Studio/Azure Machine Learning managed online endpoints (hosting the LLM personas) will be consolidated into a central Log Analytics workspace. This was configured during the setup of these services.
    *   **Container Apps (LiteLLM):** Standard output/error streams, application logs (if LiteLLM is configured for structured logging).
    *   **AI Studio Endpoints:** Logs related to model inferencing, request handling, and any custom logging implemented in scoring scripts.
*   **GitHub Actions Logs:** While primarily within GitHub, critical errors or summary statistics from GitHub Actions workflows (e.g., TDD engine outcomes) can be pushed to Log Analytics via custom scripts or Azure Functions for unified dashboarding if needed.

### 1.2. Key Metrics to Track

**A. LiteLLM (Azure Container App):**
*   **Request Rate:** Number of requests per minute/hour.
*   **Error Rate:** Percentage of requests resulting in errors (e.g., 4xx, 5xx from LiteLLM itself or downstream LLMs).
*   **Latency:** Average and percentile (p95, p99) response times for requests passing through LiteLLM.
*   **CPU and Memory Utilization:** Of the LiteLLM container instances.
*   **Network In/Out:** Data volume processed by LiteLLM.

**B. AI Personas (Azure AI Studio/ML Managed Online Endpoints):**
*   **GPU Utilization (per instance):** Critical for cost-efficiency and performance.
*   **Request Latency (per endpoint/deployment):** Time taken by the model to process a request.
*   **Deployment Status/Health:** Availability and health of each model endpoint.
*   **CPU/Memory Utilization (per instance):** For the VM instances hosting the models.
*   **Number of Requests (per endpoint/deployment):** To understand usage patterns.
*   **Provisioned Instances:** Current number of active instances (especially to verify scale-to-zero).
*   **Failed Requests:** Requests that failed at the model endpoint level.

**C. GitHub Actions (TDD Engine & Deployments):**
*   **Workflow Success/Failure Rates:** Overall reliability of the CI/CD pipelines.
*   **Workflow Duration:** Time taken for key workflows to complete.
*   **Runner Utilization (if using self-hosted runners):** CPU/memory/disk usage of runners. For GitHub-hosted, this is less of a concern beyond job minutes.
*   **Cost per Workflow Run (estimated):** Track minutes used by GitHub-hosted runners or costs associated with self-hosted runners.

### 1.3. Example Dashboard Components (Azure Dashboards)

Create a shared Azure Dashboard to visualize key metrics:

*   **Overview:**
    *   Overall system health status (up/down indicators for LiteLLM, main LLM endpoints).
    *   Total estimated cost for the current month (from Azure Cost Management).
*   **LiteLLM Performance:**
    *   Time-series charts for request rate, error rate, average latency.
    *   Gauges for current CPU/memory utilization.
*   **AI Persona Endpoints (repeated for each key persona like Pacer, Builder):**
    *   Time-series chart for GPU utilization.
    *   Time-series chart for average latency.
    *   Current number of active instances (to monitor scale-to-zero).
    *   Error rate for the endpoint.
*   **GitHub Actions Overview (if integrated):**
    *   Success/failure ratio for main workflows.
    *   Average workflow duration.
*   **Cost Overview:**
    *   Top services by cost.
    *   Burn rate against budget.

### 1.4. Alert Configurations (Azure Monitor Alerts)

Configure alerts for critical conditions to enable proactive responses:

*   **High Error Rate (LiteLLM):** If error rate > X% over Y minutes.
*   **High Latency (LiteLLM/AI Personas):** If p95 latency > Z seconds over Y minutes.
*   **Resource Exhaustion (LiteLLM/AI Personas):** If CPU/GPU utilization > X% for Y minutes (could indicate need for scaling up or optimization).
*   **Endpoint Unavailability (AI Personas):** If an endpoint is down or unhealthy.
*   **Scale-to-Zero Failure:** If an endpoint has >0 instances after being idle for longer than the configured shutdown time (custom log query might be needed).
*   **Budget Alerts (from Azure Cost Management):** If actual or forecasted costs exceed X% of the budget.
*   **GitHub Actions Failure:** If a critical workflow (e.g., main TDD pipeline, deployment) fails.

### 1.5. Application Insights (Optional but Recommended)

*   For deeper insights into LiteLLM's performance, consider integrating Azure Application Insights. This can provide distributed tracing, detailed dependency tracking (e.g., calls to each LLM endpoint), and more granular performance profiling within the LiteLLM application code itself. This would require SDK integration within the LiteLLM container.

## 2. Cost Management Strategy (Azure Cost Management + Billing)

Proactive cost management is essential to ensure the solution remains economically viable.

*   **Regular Reviews:**
    *   Schedule weekly or bi-weekly reviews of cost dashboards in Azure Cost Management.
    *   Monthly deep-dive reviews to analyze trends and identify optimization opportunities.
*   **Tagging:**
    *   Implement a consistent tagging strategy for ALL Azure resources.
    *   Example Tags: `Project: AICouncil`, `Environment: Dev/Staging/Prod`, `ServiceName: LiteLLM/PacerLLM/BuilderLLM/CoreInfra`, `Owner: TeamName`.
    *   Use tags to filter costs and group resources in cost analysis.
*   **Budgets:**
    *   Set budgets at various scopes:
        *   Overall subscription budget.
        *   Resource group budgets (e.g., for `rg-ai-services-<env>`, `rg-app-hosting-<env>`).
        *   Service-specific budgets if needed (e.g., for GPU VMs if not using managed endpoints exclusively).
    *   Configure alert notifications when actual or forecasted costs approach budget thresholds (e.g., 50%, 75%, 90%, 100%).
*   **Cost Analysis:**
    *   Utilize the cost analysis tools in Azure Cost Management to:
        *   Break down costs by service, resource group, tags, location, etc.
        *   Identify top cost drivers.
        *   Monitor cost trends over time.
        *   Detect anomalies or unexpected cost increases.
*   **Azure Advisor:**
    *   Regularly review Azure Advisor recommendations. Advisor provides actionable insights for cost optimization, performance improvement, security, and reliability, often identifying underutilized resources or better pricing tiers.

## 3. GPU Scaling/Shutdown Automation Strategy

GPU resources are typically the most expensive component. Efficient scaling and shutdown are critical.

### 3.1. Azure AI Studio Managed Online Endpoints (Scale-to-Zero)

*   **Built-in Feature:** Azure Machine Learning managed online endpoints (used by AI Studio for deploying LLM personas) support automatic scale-to-zero.
*   **Configuration:**
    *   When deploying or updating a model endpoint, set the **minimum instance count to `0`**.
    *   Configure the **`Idle shutdown` time** (e.g., `PT15M` for 15 minutes, `PT1H` for 1 hour). This is the duration of inactivity after which the deployment will scale down to zero instances.
*   **Verification:**
    *   Monitor the "Provisioned Instances" metric in Azure Monitor for the specific deployment.
    *   After ensuring no traffic is sent to the endpoint for a period longer than the configured idle shutdown time, verify that the instance count drops to `0`.
    *   The first request after scaling to zero will experience a "cold start" latency as a new instance is provisioned. This trade-off (cost vs. latency for infrequent calls) needs to be considered.

### 3.2. Manual Override or Scheduled Shutdown (for Managed Endpoints or other GPU VMs)

While scale-to-zero is preferred for managed endpoints, there might be scenarios for more direct control or for managing other GPU VMs not part of managed endpoints.

*   **Options for Control:**
    *   **Azure CLI:** `az ml online-deployment update --min-instances X --max-instances Y` (can set min to 0 or a higher number).
    *   **PowerShell:** `Update-AzMlOnlineDeployment -MinInstanceCount X -MaxInstanceCount Y`.
    *   For standalone GPU VMs: `az vm deallocate` (to stop and deallocate, stopping charges for compute) and `az vm start`.
*   **Execution Methods:**
    *   **Azure Automation:** Create runbooks (PowerShell or Python) that execute Azure CLI/PowerShell commands on a schedule (e.g., scale down non-critical dev/test endpoints outside of work hours, scale up before work hours).
    *   **GitHub Actions:** Use scheduled workflows (`on: schedule`) to run Azure CLI/PowerShell scripts. Requires setting up an Azure service principal with appropriate permissions and storing its credentials as GitHub secrets.
    *   **Azure Functions:** Time-triggered Azure Functions can execute scripts to manage GPU resources.
*   **Considerations:**
    *   **Cold Starts:** Forcing a complete shutdown (deallocation for VMs, or scaling min instances to 0 for endpoints) will always incur cold start latency when resources are restarted/rescaled. This needs to be acceptable for the workload.
    *   **Complexity:** Managing custom shutdown logic adds complexity compared to the built-in scale-to-zero feature of managed endpoints.
    *   **State Management:** Ensure that any stateful operations are handled gracefully before shutdown if dealing with standalone VMs.

### 3.3. LiteLLM Caching

*   **Cost-Saving Measure:** LiteLLM offers various caching strategies (e.g., in-memory, Redis).
*   **Benefit:** If similar prompts are frequently sent to the LLMs, LiteLLM can return cached responses instead of making new API calls to the (potentially costly) LLM endpoints. This reduces token consumption and can improve latency for cached requests.
*   **Configuration:** Configure caching directly in LiteLLM's `config.yaml` or via environment variables. Choose a cache TTL (time-to-live) appropriate for your use case.

By implementing these monitoring, cost management, and scaling strategies, the AI Council platform can operate efficiently, cost-effectively, and with high availability, providing proactive insights into its behavior and resource consumption.
```
