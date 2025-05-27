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
```
