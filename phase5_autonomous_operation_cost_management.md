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
