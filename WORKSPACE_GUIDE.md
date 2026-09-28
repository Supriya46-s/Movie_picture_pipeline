# Movie Picture CI/CD Pipeline — Workspace & Submission Guide

This guide provides the complete, step-by-step instructions to take this completed codebase, run it in your **Udacity Workspace / AWS environment**, test all CI/CD pipelines, and fulfill every project requirement.

---

## Architecture Overview

```
                          ┌────────────────────────┐
                          │   GitHub Repository    │
                          └───────────┬────────────┘
                                      │
              ┌───────────────────────┴───────────────────────┐
              ▼                                               ▼
    [Pull Request to main]                          [Push to main branch]
              │                                               │
   ┌──────────────────────┐                       ┌──────────────────────┐
   │ Frontend / Backend   │                       │ Frontend / Backend   │
   │ Continuous Integrat. │                       │ Continuous Deploym.  │
   ├──────────────────────┤                       ├──────────────────────┤
   │ 1. Lint & Test       │                       │ 1. Lint & Test       │
   │    (Runs in Parallel)│                       │ 2. Build Docker Image│
   │ 2. Conditional Build │                       │    (Tagged with SHA) │
   │    (needs: [lint,    │                       │ 3. Push to AWS ECR   │
   │     test])           │                       │ 4. Deploy to AWS EKS │
   └──────────────────────┘                       │    (Using Kustomize) │
                                                  └───────────┬──────────┘
                                                              │
                                            ┌─────────────────┴─────────────────┐
                                            ▼                                   ▼
                                  ┌───────────────────┐               ┌───────────────────┐
                                  │   AWS ECR Repos   │               │   AWS EKS Cluster │
                                  │ (frontend/backend)│               │ (Pod & LoadBal.)  │
                                  └───────────────────┘               └───────────────────┘
```

---

## Phase 1: Push Code to Your GitHub Repository

The CI/CD pipelines require a GitHub repository with GitHub Actions enabled.

### From inside the Workspace terminal:
1. Open the terminal in VS Code in the workspace.
2. Authenticate the GitHub CLI:
   ```bash
   gh auth login
   ```
   - Select: `GitHub.com`
   - Select: `HTTPS`
   - Select: `Login with a web browser`
   - Copy the one-time code shown, press **Enter**, log in, and authorize.
3. Configure your Git identity:
   ```bash
   git config --global user.email "your-email@example.com"
   git config --global user.name "Your Name"
   ```
4. Initialize the repo, commit all files, and push:
   ```bash
   git init
   git branch -M main
   git add .
   git commit -m "feat: setup full CI/CD pipeline"
   gh repo create udacity-build-cicd-project --source=. --public --push
   ```
   *(Or push to an existing remote: `git remote add origin <REPO_URL>` then `git push -u origin main`)*

---

## Phase 2: Deploy AWS Infrastructure with Terraform

The infrastructure (VPC, subnets, EKS Cluster with Bottlerocket node group, ECR repositories, and IAM roles) is provisioned using Terraform.

1. In the terminal:
   ```bash
   cd setup/terraform
   terraform init
   terraform apply -auto-approve
   ```
   *Note: Provisioning the EKS cluster and node group usually takes 10–15 minutes.*

2. After Terraform completes, inspect the outputs:
   ```bash
   terraform output
   ```
   You will see:
   - `frontend_ecr`: ECR registry URL for frontend (e.g., `XXXXXXXXXXXX.dkr.ecr.us-east-1.amazonaws.com/frontend`)
   - `backend_ecr`: ECR registry URL for backend (e.g., `XXXXXXXXXXXX.dkr.ecr.us-east-1.amazonaws.com/backend`)
   - `cluster_name`: `cluster`
   - `github_action_user_arn`: ARN for `github-action-user`

---

## Phase 3: Generate AWS IAM Access Keys for GitHub Actions

1. In your classroom, open the AWS Management Console (via Cloud Gateway).
2. Go to the **IAM** service.
3. In the left sidebar, click **Users** and select `github-action-user`.
4. Click on the **Security credentials** tab.
5. Scroll down to **Access keys** and click **Create access key**.
6. Select **Application running outside AWS** and click **Next**, then click **Create access key**.
7. Copy both the **Access key ID** and **Secret access key**.

---

## Phase 4: Authorize GitHub Actions User in Kubernetes

Before GitHub Actions can execute `kubectl` commands against the EKS cluster, the IAM user must be mapped to `system:masters` in the cluster's `aws-auth` ConfigMap.

Configure your local kubeconfig first:
```bash
aws eks update-kubeconfig --name cluster --region us-east-1
```

### Method A (Direct ConfigMap Patch):
Edit `setup/aws-auth-patch.yaml` with your Vocareum AWS Account ID (replace `ACCOUNT_ID`), then run:
```bash
kubectl apply -f ./setup/aws-auth-patch.yaml
```

### Method B (Helper Script):
```bash
cd setup
chmod +x init.sh
./init.sh
```

Verify connection:
```bash
kubectl get nodes
```
You should see 1 node in the `Ready` status.

---

## Phase 5: Configure GitHub Repository Secrets

1. In your GitHub repository, navigate to **Settings** > **Secrets and variables** > **Actions**.
2. Under **Repository secrets**, click **New repository secret** and add:

| Secret Name | Value | Purpose |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | Access key ID from Phase 3 (or Cloud Resources) | Authenticates GitHub Actions with AWS |
| `AWS_SECRET_ACCESS_KEY` | Secret access key from Phase 3 (or Cloud Resources) | Authenticates GitHub Actions with AWS |
| `AWS_SESSION_TOKEN` | Session token (if using Vocareum/AWS Academy) | Required for temporary session credentials |

---

## Phase 6: Deploy Backend via Continuous Deployment

1. On GitHub, navigate to the **Actions** tab.
2. Select **Backend Continuous Deployment** (`backend-cd.yaml`) in the left sidebar.
3. Click **Run workflow** > branch: `main` > **Run workflow** (or push a commit touching `starter/backend/`).
4. Watch the pipeline execute:
   - `Lint Backend`: Runs `pipenv run lint` (Flake8)
   - `Test Backend`: Runs `pipenv run test` (Pytest)
   - `Build, Push and Deploy Backend`: Builds Docker image, tags it with `${{ github.sha }}` and `latest`, pushes to Amazon ECR, and applies Kustomize manifests to EKS.
5. Once complete, retrieve the backend LoadBalancer external URL:
   ```bash
   kubectl get service backend -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
   ```
6. Test the backend endpoint:
   ```bash
   curl http://<BACKEND-LOADBALANCER-HOST>/movies
   ```
   Expected response:
   ```json
   {"movies":[{"id":"123","title":"Top Gun: Maverick"},{"id":"456","title":"Sonic the Hedgehog"},{"id":"789","title":"A Quiet Place"}]}
   ```

---

## Phase 7: Deploy Frontend via Continuous Deployment

1. On GitHub, navigate to the **Actions** tab > **Frontend Continuous Deployment** (`frontend-cd.yaml`).
2. Click **Run workflow** > branch: `main` > **Run workflow** (or push a commit touching `starter/frontend/`).
3. The workflow automatically discovers the backend LoadBalancer hostname from EKS via `kubectl get service backend` and injects `REACT_APP_MOVIE_API_URL` during the Docker build.
4. Watch the pipeline execute:
   - `Lint Frontend`: Runs `npm run lint` (ESLint)
   - `Test Frontend`: Runs `CI=true npm test`
   - `Build, Push and Deploy Frontend`: Builds Docker image with backend URL, tags with `${{ github.sha }}` and `latest`, pushes to ECR, and deploys to EKS.
5. Once deployed, get the frontend LoadBalancer URL:
   ```bash
   kubectl get service frontend -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
   ```
6. Open your web browser to:
   ```
   http://<FRONTEND-LOADBALANCER-HOST>
   ```
   You will see the **Movie List** catalog interface displaying movies fetched live from the backend API!

---

## Phase 8: Verify Continuous Integration (CI) and Failure Scenarios (Rubric Requirement)

The rubric requires proving that:
1. CI runs on Pull Requests against `main`.
2. Lint and Test run in parallel.
3. Build runs ONLY if Lint and Test pass (`needs: [lint, test]`).
4. Pipeline fails properly when failing tests are introduced.

### Test 1: Frontend Failure Simulation
1. Create and checkout a feature branch:
   ```bash
   git checkout -b test-frontend-failure
   ```
2. Open `starter/frontend/src/components/__tests__/App.test.js`.
   Temporarily change `'Movie List'` on line 5 to `'WRONG_HEADING'`.
3. Commit and push:
   ```bash
   git add starter/frontend/src/components/__tests__/App.test.js
   git commit -m "test: simulate frontend test failure"
   git push -u origin test-frontend-failure
   ```
4. Open a **Pull Request** to `main` on GitHub.
5. Notice that `Frontend Continuous Integration` triggers:
   - `Lint Frontend` and `Test Frontend` run in parallel.
   - `Test Frontend` fails.
   - `Build Frontend Docker Image` is blocked and does not run.
6. Fix `App.test.js` back to `'Movie List'`, commit, and push. Verify the CI pipeline turns green.

### Test 2: Backend Failure Simulation
1. Create and checkout a feature branch:
   ```bash
   git checkout -b test-backend-failure
   ```
2. In `starter/backend/test_app.py`, change line 9:
   ```python
   assert response.status_code == 500  # Will fail because endpoint returns 200
   ```
3. Commit and push:
   ```bash
   git add starter/backend/test_app.py
   git commit -m "test: simulate backend test failure"
   git push -u origin test-backend-failure
   ```
4. Open a **Pull Request** to `main` on GitHub.
5. Verify `Backend Continuous Integration` triggers:
   - `Test Backend` fails.
   - `Build Backend Docker Image` is blocked.
6. Restore line 9 to `assert response.status_code == status_code`, commit and push, and verify green status.

---

## Phase 9: Teardown AWS Resources

When you have captured your screenshots and finished evaluating your deployment:
```bash
cd setup/terraform
terraform destroy -auto-approve
```

---

## Summary Checklist for Project Submission

- [x] `.github/workflows/frontend-ci.yaml`: Runs on PR to `main` (`starter/frontend/**`), parallel lint/test, conditional Docker build.
- [x] `.github/workflows/backend-ci.yaml`: Runs on PR to `main` (`starter/backend/**`), parallel lint/test, conditional Docker build.
- [x] `.github/workflows/frontend-cd.yaml`: Runs on push to `main` (`starter/frontend/**`), tags with SHA/latest, ECR push, dynamic backend discovery, Kustomize deployment to EKS.
- [x] `.github/workflows/backend-cd.yaml`: Runs on push to `main` (`starter/backend/**`), tags with SHA/latest, ECR push, Kustomize deployment to EKS.
- [x] All workflows support `workflow_dispatch` (on-demand manual trigger).
- [x] `.dockerignore` files added to both frontend and backend to accelerate image builds.
- [x] Python base image pinned to `3.10-alpine3.17` for gcc/uwsgi compatibility.
- [x] EKS node group configured with Bottlerocket AMI for compatibility with modern Kubernetes (1.30).
- [x] `setup/aws-auth-patch.yaml` created for cluster authorization.
