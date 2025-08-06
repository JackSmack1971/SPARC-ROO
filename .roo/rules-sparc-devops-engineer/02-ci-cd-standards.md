# DevOps Engineer Rule: 02 - Secure CI/CD Pipeline Framework

**Purpose:** To define the standardized, secure, and automated pipeline for building, testing, and deploying our software. This pipeline is the primary enforcement mechanism for the SPARC methodology's quality, security, and compliance standards, serving as the artery through which all value is delivered.

---

### 1. Philosophy: The Pipeline as a Product

Our CI/CD pipeline is not just a series of scripts; it is a product in its own right, with its own requirements for security, reliability, and efficiency. It automates the enforcement of rules defined by every other SPARC role.

* **Everything as Code:** The pipeline (`.github/workflows/`), infrastructure (Terraform), and application configuration (Helm charts) are all stored in version control. There are no manual "click-ops" for deployment.
* **Immutable Artifacts:** We build a container image **once** per commit. This exact, version-tagged image is promoted through every environment (Testing, Staging, Production). This eliminates environment drift and ensures what we test is what we deploy.
* **Gated Promotions:** An artifact cannot proceed to the next stage of the pipeline without passing all quality, security, and compliance gates of the current stage. A failure stops the line, providing fast feedback.
* **Shift-Left Security & Quality:** Security is not a final step. We integrate SAST, dependency scanning, and quality checks as early as possible in the pipeline to catch issues when they are cheapest and fastest to fix.

---

### 2. Git Workflow and Pipeline Triggers

The pipeline's behavior is intrinsically linked to our Git workflow, ensuring the right checks are run at the right time.

* **On Pull Request (to `main`):** The **Continuous Integration (CI)** pipeline runs. Its goal is fast feedback (<10 minutes). It answers the question: "Is this change safe to merge?"
    * **Jobs:** Lint, Format, Unit Tests, Coverage Check, SAST, Dependency Scan, Secret Scan.
* **On Merge (to `main`):** The **Continuous Delivery (CD)** pipeline runs. It builds the final, canonical artifact and prepares it for release.
    * **Jobs:** Build & Push Docker Image, Integration Tests, Deploy to Staging.
* **On Git Tag (e.g., `v1.2.3`):** The **Continuous Deployment** pipeline runs. It promotes a previously tested artifact to production.
    * **Jobs:** Deploy to Production (with manual approval gate).

---

### 3. The Standard Pipeline-as-Code Framework

Our pipelines are defined declaratively using GitHub Actions. This provides versioning, reviewability, and reusability.

* **CI Workflow (`.github/workflows/ci-pipeline.yml`):**

```yaml
# This pipeline runs on every pull request to the main branch.
name: Continuous Integration

on:
  pull_request:
    branches: [ main ]

jobs:
  validate-code:
    name: Lint, Test, and Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with: { node-version: 18, cache: 'npm' }
      - run: npm ci

      - name: Lint and Format Check
        run: npm run lint

      - name: Unit Tests & Coverage Gate
        run: npm test -- --coverage
        # This step will fail if Jest coverageThresholds are not met.

      - name: Run Snyk to check for vulnerabilities (SAST & Dependencies)
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          command: test

      - name: Secret Detection Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.pull_request.base.sha }}
          head: ${{ github.event.pull_request.head.sha }}
````

  * **CD Workflow (`.github/workflows/cd-pipeline.yml`):**

<!-- end list -->

```yaml
# This pipeline runs on every merge to the main branch.
name: Continuous Delivery

on:
  push:
    branches: [ main ]

env:
  ECR_REPOSITORY: my-app
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-and-deploy-staging:
    name: Build and Deploy to Staging
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write # Required for AWS OIDC authentication

    steps:
      # 1. Checkout, Login to AWS, Build & Push Docker Image
      - uses: actions/checkout@v3
      - uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: arn:aws:iam::ACCOUNT_ID:role/github-actions-role
          aws-region: us-east-1
      - uses: aws-actions/amazon-ecr-login@v1
      - name: Build and push Docker image
        run: |
          docker build -t ${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }} .
          docker tag ${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }} ACCOUNT_[ID.dkr.ecr.us-east-1.amazonaws.com/$](https://ID.dkr.ecr.us-east-1.amazonaws.com/$){{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}
          docker push ACCOUNT_[ID.dkr.ecr.us-east-1.amazonaws.com/$](https://ID.dkr.ecr.us-east-1.amazonaws.com/$){{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}

      # 2. Run Integration Tests (can be a separate job)
      - name: Run Integration Tests
        run: npm run test:integration
        # This would use the newly built image via Docker Compose

      # 3. Deploy to Staging Environment
      - name: Deploy to Staging
        uses: helm/chart-releaser-action@v1.5.0
        with:
          charts_dir: ./helm
          config: .cr.yaml
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
          # Helm values would point to the new IMAGE_TAG

      # 4. Run E2E Tests against Staging
      - name: Run E2E Tests
        run: npm run test:e2e -- --browser chromium
        # Cypress/Playwright tests run against the staging URL
```

-----

### 4\. Secure Deployment Strategies

We mandate safe deployment patterns to minimize the risk and impact of production releases.

  * **Blue-Green Deployment (Default Strategy):**
      * **Process:** The new version ("Green") is deployed alongside the current production version ("Blue"). Both are running on live infrastructure. After the Green environment passes automated smoke tests, the Kubernetes service selector is updated to switch 100% of live traffic from Blue to Green.
      * **Benefit:** Zero-downtime deployment and near-instantaneous rollback (just switch the selector back to Blue).
  * **Canary Release (For High-Risk Changes):**
      * **Process:** The new version is released to a small subset of production traffic (e.g., 5%). We monitor key metrics (error rates, latency) for a predefined period. If metrics are stable, traffic is incrementally increased until it reaches 100%. If issues arise, traffic is immediately routed back to the old version.
      * **Benefit:** Limits the "blast radius" of a potentially bad release. Requires sophisticated traffic management (e.g., Istio, Linkerd).

-----

### 5\. Memory Bank Integration: Pipeline Decisions

Major architectural decisions regarding the CI/CD pipeline are documented to provide context for the "why" behind our automation.

**Documentation Triggers:**

  * A new, shared pipeline template (e.g., for a new type of service) is created.
  * A major tool is added or replaced in the pipeline (e.g., switching from Jenkins to GitHub Actions).
  * A new deployment strategy is approved and implemented.

**Memory Bank Template (`pipeline-decision.yml`)**

```yaml
#---
# File: memory/devops-engineer/pipeline-decision-blue-green-deployment.yml
#---
apiVersion: sparc.dev/v1
kind: MemoryEntry
metadata:
  domain: devops-engineer
  layer: deployment
  title: "Adoption of Blue-Green Deployment Strategy"
  author: "SPARC-DevOps-Engineer"
  timestamp: "2025-08-06T00:15:00Z"
spec:
  summary: "Standardized on Blue-Green deployment as the default strategy for all stateless microservices to achieve zero-downtime releases and instant rollback capabilities."
  type: "Pipeline Decision"
  
  content: |
    ### Problem:
    Our previous "rolling update" strategy caused brief periods of mixed versions and made rollbacks slow and complex, requiring a full redeploy of the old version.

    ### Decision:
    We will implement a Blue-Green deployment pattern using Kubernetes Services and Deployments. The CI/CD pipeline will:
    1. Deploy the new version with a "green" label.
    2. Wait for the new deployment to be fully healthy.
    3. Run automated smoke tests against the new Green service endpoint.
    4. Upon success, update the main service's `selector` to point to `color: green`.
    5. After a waiting period, scale down the old Blue deployment.

    ### Rationale:
    1.  **Zero Downtime:** Traffic is switched atomically at the network level.
    2.  **Instant Rollback:** If the Green deployment shows issues, we simply patch the service selector back to `color: blue`. This is a sub-second operation.
    3.  **Simplified Testing:** We can run a full suite of tests against the Green environment with real infrastructure before it ever receives live traffic.

  associations:
    - "tech-constraint-kubernetes"
    - "tech-constraint-helm"
  
  status: "Implemented"
```