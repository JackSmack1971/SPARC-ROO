# Security Reviewer Rule: 03 - Continuous Compliance & Control Framework

**Purpose:** To translate abstract regulatory and policy requirements (e.g., SOC 2, GDPR, PCI DSS) into a framework of concrete, automated, and continuously verifiable technical controls. This "Compliance as Code" approach ensures our systems are demonstrably secure by design.

---

### 1. Philosophy: From Policies to Pipelines

We treat compliance as an engineering discipline, not a clerical task. Our framework is built on three pillars that transform abstract policies into pipeline-enforced reality:

1.  **Define & Map:** We translate each high-level requirement from a compliance framework (like SOC 2) into a specific, unambiguous technical control.
2.  **Codify (Policy as Code):** We express these controls in a machine-readable language, primarily **Rego**, the policy language for the **Open Policy Agent (OPA)**. This decouples policy logic from application code.
3.  **Enforce & Automate:** We integrate policy enforcement directly into our CI/CD pipelines and runtime environments, making compliance checks an automated, continuous process, not a point-in-time audit.

---

### 2. The Control Mapping Framework

Every security and compliance requirement is tracked as a formal `Control` object. This creates a centralized, auditable **Control Library** that links abstract requirements to concrete implementations.

* **Control Definition (`Control.ts`):**

```typescript
// src/security/ControlFramework.ts

export type ControlFramework = 'SOC 2' | 'GDPR' | 'PCI DSS' | 'Internal';
export type ControlStatus = 'Implemented' | 'Pending' | 'Compensating' | 'Not Applicable';
export type ValidationMethod = 'Automated (OPA/Rego)' | 'Automated (Unit Test)' | 'Manual (Audit Evidence)' | 'Infrastructure Scan';

/**
 * Represents a single, verifiable security or compliance control.
 */
export interface SecurityControl {
  readonly id: string; // e.g., "AC-01" (Access Control 01)
  readonly title: string;
  readonly description: string; // The "why" behind the control.
  
  // Mapping to external frameworks
  readonly mappings: {
    framework: ControlFramework;
    requirement: string; // e.g., "CC6.1", "Art. 32"
  }[];

  // The concrete technical implementation
  readonly implementation: string;

  // How we prove the control is effective
  readonly validation: {
    method: ValidationMethod;
    evidencePointer: string; // e.g., path to Rego file, test ID, link to audit doc
  };
  
  readonly owner: 'SPARC-Architect' | 'SPARC-DevOps-Engineer' | 'SPARC-Security-Reviewer';
  status: ControlStatus;
}
````

  * **Example Control Mappings:**

| Control ID | Title                           | Framework Mapping         | Implementation                                                                 | Validation                                     |
| ---------- | ------------------------------- | ------------------------- | ------------------------------------------------------------------------------ | ---------------------------------------------- |
| **DR-01** | Data Encryption at Rest         | SOC 2 CC6.1, GDPR Art. 32 | All RDS instances and S3 buckets must have server-side encryption enabled (AES-256/KMS). | Automated (OPA check on Terraform plan).       |
| **AC-03** | Least Privilege Access          | SOC 2 CC6.2, PCI DSS 7.1  | API authorization is managed by OPA policies mapping user roles to specific endpoints. | Automated (OPA policy unit tests).             |
| **LG-02** | Audit Log Integrity             | SOC 2 CC7.2, PCI DSS 10.5 | Application audit logs are shipped to a WORM (Write-Once, Read-Many) S3 bucket. | Manual (Review of AWS IAM policies and bucket configuration). |

-----

### 3\. Policy as Code (PaC) with Open Policy Agent (OPA)

**Open Policy Agent (OPA)** is the core of our automated enforcement strategy. We use its declarative language, **Rego**, to write policies that are evaluated at critical points in our system.

  * **API Authorization:** Our API Gateway offloads authorization decisions to an OPA sidecar. It sends a JSON object representing the request, and OPA returns a simple `allow` or `deny` decision.

  * **Infrastructure Compliance:** We use `conftest` (an OPA-powered tool) in our CI pipeline to scan Terraform plans, ensuring proposed infrastructure changes comply with our policies before they are ever applied.

  * **Example Rego Policy for API Authorization (`api_authz.rego`):**

<!-- end list -->

```rego
# file: .roo/policies/rego/api_authz.rego
package api.authz

# By default, deny all requests.
default allow = false

# Input structure from API Gateway
# input: {
#   "method": "GET",
#   "path": ["users", "user-123"],
#   "token": { "sub": "user-123", "roles": ["user"] }
# }

# Allow admins to do anything.
allow {
    input.token.roles[_] == "admin"
}

# Allow users to view or update their OWN data.
allow {
    input.method == "GET"
    input.path[0] == "users"
    input.path[1] == input.token.sub
}
allow {
    input.method == "PUT"
    input.path[0] == "users"
    input.path[1] == input.token.sub
}

# Allow any authenticated user to view the public "products" catalog.
allow {
    input.method == "GET"
    input.path[0] == "products"
    is_authenticated
}

is_authenticated {
    input.token.sub
}
```

-----

### 4\. Memory Bank Integration: The Central Control Library

The Memory Bank serves as our Central Control Library. Each significant control is documented as a machine-readable artifact, creating a single source of truth for auditors and developers.

**Documentation Triggers:**

  * A new product feature requires adherence to a new compliance standard (e.g., handling payment data requires PCI DSS controls).
  * A new internal policy is established.
  * The implementation or validation method for an existing control changes.

**Memory Bank Template: Security Control (`security-control.yml`)**

```yaml
#---
# File: memory/security-reviewer/controls/DR-01_DataEncryptionAtRest.yml
#---
apiVersion: sparc.dev/v1
kind: MemoryEntry
metadata:
  domain: security-reviewer
  layer: compliance
  title: "Control DR-01: Data Encryption at Rest"
  author: "SPARC-Security-Reviewer"
  timestamp: "2025-08-06T00:05:15Z"
spec:
  summary: "Defines the technical control for ensuring all persistent data stores (databases and object storage) are encrypted at rest."
  type: "Security Control"
  
  control:
    id: "DR-01"
    title: "Data Encryption at Rest"
    description: "To protect against unauthorized data access in the event of a physical storage compromise, all customer data stored at rest must be encrypted using strong, industry-standard cryptographic algorithms."
    
    mappings:
      - framework: "SOC 2"
        requirement: "CC6.1 - In-scope system data is protected from unauthorized access and use."
      - framework: "GDPR"
        requirement: "Article 32 - Security of processing"

    implementation: "All AWS RDS instances MUST be configured with `storage_encrypted = true`. All S3 buckets used for customer data MUST have `server_side_encryption_configuration` enabled with AES-256."

    validation:
      method: "Automated (OPA/Rego)"
      evidencePointer: ".roo/policies/rego/aws_storage_encryption.rego"
      
    owner: "SPARC-DevOps-Engineer"
    status: "Implemented"

```