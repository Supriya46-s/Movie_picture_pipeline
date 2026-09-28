# Project Screenshots Guide

If you wish to include screenshots with your submission (or for your portfolio), save them in this folder (`screenshots/`) and reference them in your submission:

### Recommended Screenshots:

1. **`01_backend_ci_success.png`**:
   - Screenshot of the **Backend Continuous Integration** workflow run on GitHub showing green checkmarks for `Lint`, `Test`, and `Build`.

2. **`02_backend_ci_failure.png`**:
   - Screenshot of a Pull Request where backend tests intentionally failed (e.g., modifying `test_app.py`), showing the `Test` job failed and blocked the `Build` job.

3. **`03_frontend_ci_success.png`**:
   - Screenshot of the **Frontend Continuous Integration** workflow run on GitHub showing green checkmarks for `Lint`, `Test`, and `Build`.

4. **`04_frontend_ci_failure.png`**:
   - Screenshot of a Pull Request where frontend tests intentionally failed (e.g., `FAIL_TEST=true`), showing `Test` failed and blocked `Build`.

5. **`05_backend_cd_success.png`**:
   - Screenshot of **Backend Continuous Deployment** workflow completing all jobs (`Lint`, `Test`, `Build and push image`, `Deploy to EKS`).

6. **`06_frontend_cd_success.png`**:
   - Screenshot of **Frontend Continuous Deployment** workflow completing all jobs (`Lint`, `Test`, `Build and push image`, `Deploy to EKS`).

7. **`07_kubernetes_resources.png`**:
   - Terminal screenshot showing:
     ```bash
     kubectl get pods -A
     kubectl get services -n default
     ```

8. **`08_movie_picture_live_browser.png`**:
   - Browser screenshot visiting the Frontend LoadBalancer URL (`http://<FRONTEND-EXTERNAL-IP>`) showing the live Movie Picture web catalog!
