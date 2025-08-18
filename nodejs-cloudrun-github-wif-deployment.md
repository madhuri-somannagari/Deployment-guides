# Secure Deployment of Node.js Application to Cloud Run using GitHub Actions & Workload Identity Federation

### 1.Overview
Deploying a node.js app securely to Cloud Run via GitHub Actions. Using Workload Identity Federation (WIF) for secure authentication (no service account keys).
### 2. Prerequisites
Before starting:
1.	You have a Node.js app (like server.js) working locally.
2.	You have Owner or similar permissions on the Google Cloud project.
3.	GitHub repo is connected and working.
### 3. Project folder Structure
```
project/
├── server.js
├── Dockerfile
├── .github/
│   └── workflows/
│       └── deploy.yml
├── package.json
```
### 4. Docker file
A Dockerfile is a text file that contains a set of instructions used to automate the creation of a Docker image. It defines how to build the environment where your application will run — including the base OS, application code, dependencies, configurations, and commands to start the service.
We use the dockerfile for consistency, automation, portability, version control, reproduceability, isolation. 
Write a dockerfile in the project root: `repo_dir/Dockerfile`

``` docker
# ---------- Stage 1: Builder ----------
FROM node:18-alpine AS builder

# Set working directory
WORKDIR /app

# Install dependencies ( if we use npm avoid yarn)
COPY package.json package-lock.json ./ 
RUN npm ci --omit=dev

# Copy application code
COPY . .

# ---------- Stage 2: Production Image ----------
FROM node:18-alpine

# Set environment
ENV NODE_ENV=production
WORKDIR /app

# Copy only the necessary files from builder
COPY --from=builder /app/package.json ./
COPY --from=builder /app/package-lock.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/server.js ./
COPY --from=builder /app/*.jsx ./
COPY --from=builder /app/public ./public
COPY --from=builder /app/babel.config.js ./

# Expose application port
EXPOSE 3000
#EXPOSE 8080

# Start the application
CMD ["node", "server.js"]
```

### 5. Google Cloud Setup
#### 5.1 API – application Programming Interface 
These APIs let you create, update, delete, or access resources programmatically. Without enabling an API, your code or automation can’t use that service.
Imagine Google Cloud is a city with locked buildings (services).
APIs are the front doors. You unlock a door (enable an API) before entering and using what's inside (like deploying apps or storing images).
Enable APIs
Go to ` Google Cloud Console > APIs & Services > Library `
Enable:
1.	Cloud Run API
2.	IAM Credentials API
3.	Artifact Registry API

#### 5.2 Service account
This account will be impersonated by GitHub via WIF to deploy your app.
Create Service Account for GitHub to Access GCP
1.	Go to ` IAM & Admin > Service Accounts `
•	Details: `Name: github-deployer`, `ID: github-deployer`, `Description: GitHub Actions deployer`
2.	Click Create & Continue
3.	Assign roles:
•	Cloud Run Admin (roles/run.admin)
•	Service Account User (roles/iam.serviceAccountUser)
4.	Click Done
5.	Make sure GitHub SA has permission to invoke the Cloud Run service and User.

#### 5.3 Enable Workload Identity Federation (WIF)

WIF to impersonate a GCP Service Account and deploy your app to Cloud Run securely.
This setup provides a secure, automated deployment pipeline from GitHub to Cloud Run without managing long-lived secrets.
***Real-world Analogy*** - Imagine Cloud Run is a secure warehouse, and deploying your app is like delivering a package. Instead of handing over a master key (SA key file), GitHub shows a verified badge (OIDC token) to get access. GCP checks the badge against a trusted database (WIF), then opens the door.
***Benefits***
•	Fully automated deployments from main branch
•	Strong identity validation (OIDC + WIF)
•	Minimal credential management
•	Follows Google Cloud security best practices

***Create WIF Pool***
1.	Go to `IAM & Admin > Workload Identity Federation > Create Pool`:
o	Name: `github-pool`
o	Location: `global`
o	Description: `GitHub Pool`
2.	Click Continue, then Create
Create WIF ODIC Provider
3. Inside the pool github-pool, click “Add Provider”
o	Name: `github`
o	Type: `OIDC`
o	Issuer: `https://token.actions.githubusercontent.com`
o	Attribute Mapping:
```google.subject = assertion.sub
attribute.repository = assertion.repository
```
4. click create

Bind GitHub to the service account:
This binds your GitHub repo to GCP via WIF, allowing it to act as the SA.

1.	Go to `IAM & Admin > Service Accounts`
2.	Find `service_name@[PROJECT_ID].iam.gserviceaccount.com`
3.	Click the `email > Permissions tab`
4.	Click “Grant Access”
•	Member:
```
principalSet://iam.googleapis.com/projects/[PROJECT_NUM]/locations/global/workloadIdentityPools/[github-pool-ID]/attribute.repository/[GITHUB_USER]/[REPO_NAME]
```
•	Role: Workload Identity User `(roles/iam.workloadIdentityUser)`

### 6. Artifact Registry
A secure storage location for Docker images and other build artifacts. Like DockerHub or GitHub Container Registry -- but private, region-based, and GCP-native.
Build the Docker Image Inside the GitHub Actions or local, tag the image properly and push it to docker image. Before that pushing github must authenticate with GCP
` docker build -t ${{ secrets.REGION }}-docker.pkg.dev/${{ secrets.PROJECT_ID }}/${{ secrets.REPO_NAME }}/node-app:latest . `
` docker push ${{ secrets.REGION }}-docker.pkg.dev/${{ secrets.PROJECT_ID }}/${{ secrets.REPO_NAME }}/node-app:latest `

###### Create Artifact Registry repo
Go to `Cloud Console > Artifact Registry > Repositories`
Click "Create Repository"
1.	Name: `my-repo`
2.	Format: `Docker`
3.	Region: `(e.g., asia-south1)`
4.	Click Create

### 7. GitHub Setup – Secrets
GitHub Actions is GitHub’s built-in Continuous Integration/Continuous Deployment (CI/CD) platform. It automates our build → test → deploy pipeline, allowing developers to focus on writing code instead of manual deployments or scripting.
***Problem It Solves***
•	No need for a separate CI/CD platform
•	Reduces manual steps in code deployment
•	Ensures consistent, repeatable builds
•	Enables secure identity-based deployments with WIF
***Benefits***
•	****Native to GitHub****: No additional tooling needed
•	****Event-driven****: Automatically triggers on push, PRs, etc.
•	****Secure****: Supports OIDC/WIF instead of key-based auth
•	****Modular****: Reusable Actions from GitHub Marketplace

In your GitHub repo, ` go to Settings > Secrets and Variables > Actions, and add the following secrets `:
| Name         | Value                                                                 |
|--------------|----------------------------------------------------------------------|
| PROJECT_ID   | Your GCP project ID                                                  |
| REPO_NAME    | my-repo (Artifact Registry name)                                     |
| REGION       | e.g., asia-south1                                                     |
| GCP_SA_EMAIL | Service_name@[PROJECT_ID].iam.gserviceaccount.com                    |
| WIF_PROVIDER | projects/[PROJECT_NUM]/locations/global/workloadIdentityPools/github-pool/providers/github |
| RUNTIME_SA   | The runtime service account used by Cloud Run (same or separate from deployer SA) |

***In repo add file .yaml***
The structure in the repo to add yaml file `.github/workflows/deploy.yml`
Yaml file:
```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
  

jobs:
  node-server-deploy-build:
    runs-on: ubuntu-latest

    permissions:
      id-token: write   # REQUIRED for Workload Identity Federation
      contents: read

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Authentication to google Cloud
      id: auth
      uses: google-github-actions/auth@v2
      with:
        token_format: 'access_token'
        workload_identity_provider: 'projects/981221757308/locations/global/workloadIdentityPools/github-pool/providers/github'
        service_account: 'github@hdip-platform.iam.gserviceaccount.com'

    - name: Set up gcloud CLI
      uses: google-github-actions/setup-gcloud@v1
      with:
        project_id: ${{ secrets.PROJECT_ID }}
        #install_components: 'beta'

    - name: Configure Docker for Artifact Registry
      run: gcloud auth configure-docker ${{ secrets.REGION }}-docker.pkg.dev
     
    - name: Build & Push Docker image
      run: |
        docker build -t ${{ secrets.REGION }}-docker.pkg.dev/${{ secrets.PROJECT_ID }}/${{ secrets.REPO_NAME }}/node-app:latest .
        docker push ${{ secrets.REGION }}-docker.pkg.dev/${{ secrets.PROJECT_ID }}/${{ secrets.REPO_NAME }}/node-app:latest


    - name: Deploy to Cloud Run
      run: |
        gcloud run deploy node-app \
          --image=${{ secrets.REGION }}-docker.pkg.dev/${{ secrets.PROJECT_ID }}/${{ secrets.REPO_NAME }}/node-app:latest \
          --platform=managed \
          --region=${{ secrets.REGION }} \
          --no-allow-unauthenticated
      #--port=3000
    
```
### 8. Workflow (Step-by-Step)
CI/CD Pipeline Steps
- Developer pushes to main
- GitHub Actions triggers deploy.yml workflow
- GitHub authenticates using OIDC token → verified by GCP’s WIF provider
- GCP grants a short-lived token to GitHub to act as the github-deployer service account
- Docker image is built and pushed to Artifact Registry
- New version of app is deployed to Cloud Run
- Cloud Run uses the runtime service account for execution permissions

Key Files
-	.github/workflows/deploy.yml
-	Dockerfile
-	package.json, server.js
-	.dockerignore

### 10. Principles / Best Practices

- 🔐 Least Privilege: Service accounts should only have roles necessary for their job
- 🚫 No Key Files: Use WIF over key files for security
- 📛 Naming Conventions:
SA: github-deployer@[PROJECT_ID].iam.gserviceaccount.com
Pool: github-pool, Provider: github
- 🚧 Separate Prod & Staging: Never reuse prod identities in staging/dev
- 🔄 Short-lived Access: WIF ensures short-lived credentials

### 11. Troubleshooting

| Symptom                                   | Possible Cause                     | Fix                                                                    |
|-------------------------------------------|-------------------------------------|-------------------------------------------------------------------------|
| 403 PERMISSION_DENIED in deploy step      | WIF not correctly bound to SA       | Recheck trust binding for GitHub repo in WIF                            |
| Docker push to Artifact Registry fails    | Missing Docker auth                 | Ensure `gcloud auth configure-docker` runs before push                  |
| Workflow not triggering                   | Wrong branch/event defined in `on:` block | Make sure workflow branch and event match your intended triggers |

### 12. Logging / Debugging Tips
Check Actions `tab → Select job → View logs step-by-step`
Use gcloud run services describe <service> to check deployment status
Use gcloud iam workload-identity-pools list and describe to check mappings
GitHub Actions job logs show which steps fail and can emit WIF debug info
Check cloud run logs, check service account roles and permissions.

### 13. Links & References
- [Official Docs](https://docs.github.com/en/actions)
- [GitHub Actions Docs](https://docs.github.com/en/actions)
- [Google Cloud Docs](https://cloud.google.com)
- [Workload Identity Federation](https://cloud.google.com/blog/products/identity-security/enabling-keyless-authentication-from-github-actions)



