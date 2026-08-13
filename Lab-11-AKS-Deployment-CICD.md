# Lab 11: Implement GitHub Actions for CI/CD using AKS

## Objective

In this lab, you will implement a simple CI/CD pipeline using **GitHub Actions**, **Azure Container Registry (ACR)**, and **Azure Kubernetes Service (AKS)**.

The pipeline will:

1. Build the eShopOnWeb application.
2. Run automated tests.
3. Build a Docker image.
4. Push the image to Azure Container Registry.
5. Deploy the application to an existing AKS cluster.
6. Expose the application using an Azure Load Balancer.

## Prerequisites

- Azure subscription
- Existing AKS cluster
- Azure Cloud Shell access
- GitHub account
- Permission to create an Azure service principal and ACR

---

# Step 1: Import eShopOnWeb to Your GitHub Repository

1. Go to your **GitHub account**.
2. Go to **Repositories** and click **New**.
3. On the **Create a new repository** page, click **Import a repository** below the page title.
4. Use the following details:

| Setting | Value |
|---|---|
| Source repository URL | `https://github.com/MicrosoftLearning/eShopOnWeb` |
| Owner | Your GitHub account alias |
| Repository Name | `eShopOnWeb` |
| Privacy | Public |

5. Click **Begin Import**.
6. Wait for the repository to be ready.

---

# Step 2: Allow GitHub Actions

1. Open the **eShopOnWeb** repository.
2. Go to **Settings → Actions → General**.
3. Select **Allow all actions and reusable workflows**.
4. Click **Save**.

---

# Step 3: Set Up GitHub Repository and Azure Access

Open **Azure Cloud Shell**, select **Bash**, and execute:

```bash
az ad sp create-for-rbac   --name GH-Action-eshoponweb   --role contributor   --scopes /subscriptions/SUBSCRIPTION-ID/resourceGroups/RESOURCE-GROUP   --sdk-auth
```

Replace:

- `SUBSCRIPTION-ID` with your Azure subscription ID.
- `RESOURCE-GROUP` with your resource group name.

Both values can be found on the **Overview** page of the Resource Group.

---

# Step 4: Copy the Service Principal JSON

The command will output a JSON object.

Copy the **complete JSON object** and keep it ready for the next step.

The JSON contains the identifiers used to authenticate against Azure using the Microsoft Entra service principal.

---

# Step 5: Add the Azure Credentials as a GitHub Secret

1. Open the **eShopOnWeb** GitHub repository.
2. Go to **Settings → Secrets and variables → Actions**.
3. Click **New repository secret**.
4. Enter:

**Name:**

```text
AZURE_CREDENTIALS
```

**Secret:**

Paste the complete JSON object copied in Step 4.

5. Click **Add secret**.

GitHub Actions can now reference the Azure service principal using:

```yaml
${{ secrets.AZURE_CREDENTIALS }}
```

---

# Step 6: Create an Azure Container Registry

Open Azure Cloud Shell and run:

```bash
az acr create --resource-group YOUR-RG-NAME --name ADD-UNIQUE-NAME --sku Basic
```

Replace:

- `YOUR-RG-NAME` with your resource group name.
- `ADD-UNIQUE-NAME` with a unique ACR name.

Example:

```bash
az acr create --resource-group aks-labs --name akslabacr2026 --sku Basic
```

---

# Step 7: Create the GitHub Actions Workflow

1. In your **eShopOnWeb** repository, create:

```text
.github/workflows/eshoponweb-aks-cicd.yml
```

2. Copy and paste the following workflow.
3. Replace `YOUR-RG-NAME`, `YOUR-AKS-CLUSTER-NAME`, and `YOUR_ACR_NAME` with your Azure values.

```yaml
name: eShopOnWeb Build Test and Deploy to AKS

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  RESOURCE-GROUP: YOUR-RG-NAME
  AKS-CLUSTER-NAME: YOUR-AKS-CLUSTER-NAME
  ACR-NAME: YOUR_ACR_NAME
  IMAGE-NAME: eshoponweb
  IMAGE-TAG: ${{ github.sha }}

jobs:
  buildandtest:
    runs-on: ubuntu-latest

    steps:

      # 1. Checkout source code
      - name: Checkout source code
        uses: actions/checkout@v5

      # 2. Setup .NET
      - name: Setup .NET
        uses: actions/setup-dotnet@v5
        with:
          dotnet-version: "8.0.x"

      # 3. Build application
      - name: Build with dotnet
        run: dotnet build ./eShopOnWeb.sln --configuration Release

      # 4. Run tests
      - name: Test with dotnet
        run: dotnet test ./eShopOnWeb.sln --configuration Release

      # 5. Create Dockerfile
      - name: Create Dockerfile
        run: |
          cat > Dockerfile <<'EOF'
          FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

          WORKDIR /src

          COPY . .

          RUN dotnet restore ./eShopOnWeb.sln

          RUN dotnet publish ./src/Web/Web.csproj               -c Release               -o /app/publish               /p:UseAppHost=false

          FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final

          WORKDIR /app

          COPY --from=build /app/publish .

          EXPOSE 8080

          ENV ASPNETCORE_URLS=http://+:8080

          ENTRYPOINT ["dotnet", "Web.dll"]
          EOF

      # 6. Login to Azure
      - name: Azure Login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      # 7. Build Docker image and push to ACR
      - name: Build and push image to ACR
        uses: azure/cli@v2
        with:
          inlineScript: |
            az acr build               --registry ${{ env.ACR-NAME }}               --image ${{ env.IMAGE-NAME }}:${{ env.IMAGE-TAG }}               --file Dockerfile               .

      # 8. Create Kubernetes manifest
      - name: Create Kubernetes deployment manifest
        run: |
          mkdir -p k8s

          cat > k8s/eshoponweb.yaml <<EOF
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: eshoponweb
          spec:
            replicas: 2
            selector:
              matchLabels:
                app: eshoponweb
            template:
              metadata:
                labels:
                  app: eshoponweb
              spec:
                containers:
                  - name: eshoponweb
                    image: ${{ env.ACR-NAME }}.azurecr.io/${{ env.IMAGE-NAME }}:${{ env.IMAGE-TAG }}
                    ports:
                      - containerPort: 8080
                    env:
                      - name: ASPNETCORE_ENVIRONMENT
                        value: "Docker"
                      - name: ASPNETCORE_URLS
                        value: "http://+:8080"
                      - name: baseUrls__webBase
                        value: "http://localhost:8080/"
                      - name: baseUrls__apiBase
                        value: "http://localhost:8080/api/"
                      - name: UseOnlyInMemoryDatabase
                        value: "true"
          ---
          apiVersion: v1
          kind: Service
          metadata:
            name: eshoponweb-service
          spec:
            type: LoadBalancer
            selector:
              app: eshoponweb
            ports:
              - port: 80
                targetPort: 8080
          EOF

      # 9. Connect to AKS
      - name: Set AKS context
        uses: azure/aks-set-context@v4
        with:
          resource-group: ${{ env.RESOURCE-GROUP }}
          cluster-name: ${{ env.AKS-CLUSTER-NAME }}

      # 10. Deploy application to AKS
      - name: Deploy application to AKS
        uses: Azure/k8s-deploy@v5
        with:
          namespace: default
          manifests: |
            k8s/eshoponweb.yaml
          images: |
            ${{ env.ACR-NAME }}.azurecr.io/${{ env.IMAGE-NAME }}:${{ env.IMAGE-TAG }}
          pull-images: false

      # 11. Verify deployment
      - name: Verify deployment
        run: |
          kubectl rollout status deployment/eshoponweb --timeout=180s
          kubectl get pods
          kubectl get service eshoponweb-service
```

Save and commit the workflow file to the `main` branch.

---

# Step 8: Run the Pipeline

### Option 1: Push a Change

Commit and push the workflow file to the `main` branch. GitHub Actions will automatically start the pipeline.

### Option 2: Run Manually

1. Go to your GitHub repository.
2. Click **Actions**.
3. Select **eShopOnWeb Build Test and Deploy to AKS**.
4. Click **Run workflow**.
5. Select the `main` branch.
6. Click **Run workflow**.

The pipeline performs:

```text
Checkout
   ↓
Build
   ↓
Test
   ↓
Create Docker Image
   ↓
Push Image to ACR
   ↓
Connect to AKS
   ↓
Deploy Application
   ↓
Verify Deployment
```

Wait for all workflow steps to complete successfully.

---

# Step 9: Verify the Deployment in AKS

After the pipeline completes successfully, open Azure Cloud Shell.

If required, connect to your AKS cluster:

```bash
az aks get-credentials   --resource-group YOUR-RG-NAME   --name YOUR-AKS-CLUSTER-NAME
```

### Check Pods

```bash
kubectl get pods
```

Expected result:

```text
NAME                          READY   STATUS    RESTARTS   AGE
eshoponweb-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
eshoponweb-xxxxxxxxxx-yyyyy   1/1     Running   0          1m
```

### Check Deployment

```bash
kubectl get deployment
```

Expected result:

```text
NAME         READY   UP-TO-DATE   AVAILABLE
eshoponweb   2/2     2            2
```

### Check Service

```bash
kubectl get service
```

Look for:

```text
NAME                 TYPE           EXTERNAL-IP
eshoponweb-service   LoadBalancer   <EXTERNAL-IP>
```

---

# Step 10: Access the Application

Copy the `EXTERNAL-IP` from:

```bash
kubectl get service eshoponweb-service
```

Open a web browser and enter:

```text
http://<EXTERNAL-IP>
```

Example:

```text
http://20.x.x.x
```

You should see the **eShopOnWeb application**.

---

# Lab Summary

In this lab, you implemented a complete CI/CD workflow using:

- **GitHub** – Source code repository
- **GitHub Actions** – CI/CD automation
- **.NET** – Build and test application
- **Docker** – Containerize the application
- **Azure Container Registry** – Store the container image
- **AKS** – Run the containerized application
- **Kubernetes Deployment** – Manage application Pods
- **Kubernetes Service** – Expose the application
- **Azure Load Balancer** – Provide external access

Overall flow:

```text
GitHub Repository
       |
       v
GitHub Actions
       |
       +---- Build
       |
       +---- Test
       |
       +---- Docker Build
       |
       v
Azure Container Registry
       |
       | Container Image
       v
Azure Kubernetes Service
       |
       v
Kubernetes Deployment
       |
       +--------+--------+
       |                 |
       v                 v
    Pod 1             Pod 2
       |                 |
       +--------+--------+
                |
                v
      LoadBalancer Service
                |
                v
             Internet
                |
                v
          eShopOnWeb

# End of Lab
