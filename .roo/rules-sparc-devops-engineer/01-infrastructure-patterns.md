# DevOps Engineer Rule: 01 - Infrastructure-as-Code (IaC) Framework

**Purpose:** To establish the principles and patterns for defining, deploying, and managing our cloud infrastructure using code. By treating infrastructure as a software project, we ensure it is repeatable, auditable, secure, and scalable by design, forming the bedrock upon which all applications are built.

---

### 1. Philosophy: Infrastructure as a Solved Problem

Our infrastructure is not a collection of manually configured servers; it is a version-controlled, automated system. We adhere to three core principles:

1.  **Declarative State:** We define the **desired state** of our infrastructure in code. The tooling's job is to make reality match that state. We don't write scripts that say *how* to create a server; we write manifests that declare *that* a server should exist with a specific configuration.
2.  **Immutability:** We do not modify running infrastructure. To apply a patch or change configuration, we build a new, versioned artifact (a container image or machine image), deploy it, and terminate the old one. This completely eliminates configuration drift and makes rollbacks trivial. 
3.  **Idempotency:** Our infrastructure code, written in Terraform, must be idempotent. Applying the same code multiple times must result in the exact same state, without errors or side effects. This makes infrastructure changes predictable and safe to automate.

---

### 2. Core IaC Tooling and Repository Structure

We standardize on a specific set of tools and a modular structure to manage our IaC codebase effectively.

* **Primary Tooling:**
    * **Terraform:** For provisioning cloud resources (VPCs, databases, Kubernetes clusters, IAM roles).
    * **Packer:** For building immutable Amazon Machine Images (AMIs) for our Kubernetes nodes.
    * **Helm:** For packaging and deploying applications onto our Kubernetes clusters.
* **Repository Structure:** Our Terraform code is organized into a modular, multi-environment structure to promote reuse and clarity.

    ```
    terraform/
    ├── modules/              # Reusable, versioned modules for our infrastructure primitives.
    │   └── vpc/
    │       ├── main.tf
    │       ├── variables.tf
    │       └── outputs.tf
    ├── environments/         # Environment-specific compositions of our modules.
    │   ├── _shared/          # Shared resources like IAM roles.
    │   ├── staging/
    │   │   ├── main.tf
    │   │   └── terraform.tfvars
    │   └── production/
    │       ├── main.tf
    │       └── terraform.tfvars
    └── backend.tf          # Defines the remote state backend (S3 + DynamoDB).
    ```
* **State Management:** Terraform state **MUST** be stored in a remote backend (AWS S3) with state locking enabled (DynamoDB). This is critical for team collaboration and preventing concurrent modifications.

---

### 3. Key Infrastructure Patterns

Our infrastructure is built from a set of standardized, secure, and battle-tested patterns.

#### 3.1. Secure VPC Design
Our network topology is designed with a "deny-by-default" posture.

* **Layout:** A standard three-tier design:
    * **Public Subnets:** Contain only load balancers and NAT gateways. No application compute resides here.
    * **Private Subnets:** Contain our application workloads (Kubernetes nodes, services). They can initiate outbound traffic via the NAT gateway but cannot be reached directly from the internet.
    * **Data Subnets:** Highly isolated subnets for our databases, with network ACLs that only permit traffic from the private application subnets.
* **Security Groups:** All security groups **MUST** have a default `deny all` ingress and egress policy. Traffic is only permitted via explicit, narrowly scoped rules.

    ```hcl
    # Example Terraform for a secure database security group
    resource "aws_security_group" "database_sg" {
      name        = "database-sg"
      description = "Allow traffic from application layer"
      vpc_id      = module.vpc.vpc_id

      # Deny all egress by default
      egress = []

      # Allow ingress only from the application security group on the postgres port
      ingress {
        from_port       = 5432
        to_port         = 5432
        protocol        = "tcp"
        security_groups = [aws_security_group.application_sg.id]
      }
    }
    ```

#### 3.2. Ephemeral Preview Environments
To enable high-fidelity testing and review, we automatically spin up a complete, isolated environment for every pull request.

* **Workflow:**
    1.  A developer opens a pull request.
    2.  A GitHub Actions workflow is triggered.
    3.  The workflow uses Terraform to create a new, temporary workspace and applies a configuration that creates a dedicated namespace in our staging Kubernetes cluster.
    4.  It deploys the feature branch's container image to this new namespace.
    5.  A comment with a unique URL (e.g., `pr-123.staging.our-app.com`) is posted back to the pull request.
    6.  When the pull request is merged or closed, a cleanup job runs `terraform destroy -workspace=pr-123` to tear down all resources.

#### 3.3. Cost Governance and Tagging
Cloud resources must be managed responsibly to control costs.

* **Tagging Mandate:** All provisionable resources **MUST** be tagged with `Owner`, `Project`, and `Environment`. This is enforced by a `conftest` OPA policy in the CI pipeline that fails if tags are missing from the Terraform plan.
* **Automated Shutdown:** A scheduled Lambda function runs at 8 PM EST daily to shut down all resources tagged with `Environment=staging` or `Environment=preview`.

---

### 4. Memory Bank Integration: Infrastructure ADRs

Major infrastructure decisions that have a long-term impact on cost, security, or architecture are documented as Architectural Decision Records (ADRs) in the Memory Bank.

**Memory Bank Template: Infrastructure ADR (`infra-adr.yml`)**

```yaml
#---
# File: memory/devops-engineer/adrs/2025-08-01_EKS_Adoption.yml
#---
apiVersion: sparc.dev/v1
kind: MemoryEntry
metadata:
  domain: devops-engineer
  layer: architecture
  title: "ADR-001: Adopt AWS EKS as the Standard Container Orchestrator"
  author: "SPARC-Architect"
  timestamp: "2025-08-01T10:00:00Z"
spec:
  summary: "This record documents the decision to standardize on AWS Elastic Kubernetes Service (EKS) for container orchestration, following an evaluation of available options."
  type: "Architectural Decision Record"
  
  content: |
    ### Context:
    Our move to a microservices architecture requires a robust, scalable, and resilient platform for deploying and managing containerized applications. We needed a solution that would reduce operational overhead while providing the power and flexibility of the Kubernetes ecosystem.

    ### Options Considered:
    1.  **AWS EKS (Managed Kubernetes):** A managed Kubernetes control plane provided by AWS.
    2.  **Self-Managed Kubernetes (kops/kubeadm):** Running and managing our own Kubernetes clusters on EC2 instances.
    3.  **AWS ECS (Elastic Container Service):** AWS's proprietary container orchestrator.

    ### Decision:
    We have chosen to adopt **AWS EKS** as our standard platform.

    ### Rationale:
    - **Reduced Operational Overhead:** EKS manages the availability, scalability, and patching of the Kubernetes control plane, allowing our team to focus on applications, not cluster administration.
    - **Ecosystem Compatibility:** Provides a fully conformant Kubernetes API, ensuring we can leverage the vast open-source ecosystem of tools (Helm, Prometheus, Istio, etc.) without vendor lock-in.
    - **Strong AWS Integration:** Natively integrates with IAM for authentication, VPC for networking, and ELB for load balancing, providing a secure and seamless experience.
    - **Superior to ECS:** While ECS is simpler, EKS provides greater flexibility and avoids proprietary AWS lock-in for our core orchestration logic.
    - **Superior to Self-Managed:** The operational cost and complexity of managing a secure, highly-available control plane ourselves was deemed too high.

  associations:
    - "tech-constraint-kubernetes"
    - "tech-constraint-aws"
  
  status: "Decided"
````
