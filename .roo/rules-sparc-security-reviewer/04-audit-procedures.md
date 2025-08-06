# Security Reviewer Rule: 04 - Continuous Audit and Assurance Framework

**Purpose:** To define the procedures for continuous self-auditing, automated evidence collection, and engagement with external security assessors. This framework moves us from point-in-time audits to a state of **Continuous Assurance**, where our security posture is perpetually validated.

---

### 1. Philosophy: Audit as Continuous Feedback

We treat auditing not as a disruptive event, but as a continuous feedback loop that validates and improves our security controls. The goal is to be **"always audit-ready."**

* **Automate Evidence, Audit the Automation:** Our primary strategy is to automate the collection of evidence that our controls are effective. The role of the human auditor (both internal and external) shifts from manual inspection to reviewing the automated evidence and auditing the automation itself. This is exponentially more efficient and reliable.
* **Traceability is Non-Negotiable:** Our framework is built on a "golden thread" of traceability. An auditor must be able to follow a link from a high-level requirement (e.g., SOC 2 CC6.1) in our Control Library directly to the CI/CD job that enforces it, the Rego policy that defines it, and the test cases that validate it.

---

### 2. The Audit Lifecycle: A Multi-Layered Approach

Our assurance activities operate on different cadences to provide layered, defense-in-depth validation.

#### Layer 1: Continuous Monitoring (Real-time)
* **What:** Automated security and compliance checks integrated directly into the developer workflow and CI/CD pipeline.
* **Examples:**
    * SAST scans on every commit (Snyk, SonarQube).
    * Infrastructure compliance checks on Terraform plans (`conftest`).
    * Dependency vulnerability scans (`npm audit`).
    * Coverage and mutation testing quality gates.
* **Output:** Real-time feedback to developers and a perpetually updated security dashboard.

#### Layer 2: Internal Audits (Quarterly)
* **What:** A formal, focused review led by the **SPARC-Security-Reviewer** on a specific domain (e.g., Q1: Access Control, Q2: Data Encryption, Q3: Incident Response).
* **Activities:**
    1.  Review the automated evidence collected from the Continuous Monitoring layer.
    2.  Perform manual validation of controls that cannot be fully automated (e.g., reviewing business process documentation).
    3.  Re-evaluate existing threat models for the audited domain to ensure they are current.
    4.  Produce a formal `Audit Record` in the Memory Bank, including any findings.

#### Layer 3: External Engagements (Annual & Ad-Hoc)
* **What:** Engagements with independent, third-party assessors.
* **Types:**
    * **Formal Audits:** For certifications like SOC 2 or ISO 27001.
    * **Penetration Testing:** We commission a third-party firm annually to perform black-box and grey-box penetration tests against our production environment.
* **Process:** The Security-Reviewer acts as the primary liaison, providing auditors with read-only access to our Memory Bank (Control Library and past Audit Records) and automated evidence dashboards.

---

### 3. Automated Evidence Collection

We maintain a script that aggregates compliance evidence from our various systems into a single, human-readable report. This script is the foundation of our "always audit-ready" state.

```bash
#!/bin/bash
# file: scripts/generate-assurance-report.sh

# This script generates a markdown report summarizing the status of automated controls.
# In a real implementation, it would use APIs to fetch live data.

echo "## Continuous Assurance Report - $(date)"
echo ""
echo "### Control Validation Summary"
echo ""
echo "| Control ID | Title | Validation Method | Last Run | Status | Evidence |"
echo "|------------|-------|-------------------|----------|--------|----------|"

# Example for an OPA/Conftest check
TF_LINT_STATUS=$(get_ci_job_status "tf-conftest") # Placeholder function
TF_LINT_URL=$(get_ci_job_url "tf-conftest")
echo "| DR-01 | Data Encryption at Rest | Automated (OPA/Rego) | $(date -I) | $TF_LINT_STATUS | [Link]($TF_LINT_URL) |"

# Example for a test-based check
AUTHZ_TEST_STATUS=$(get_ci_job_status "api-authz-tests")
AUTHZ_TEST_URL=$(get_ci_job_url "api-authz-tests")
echo "| AC-03 | Least Privilege Access | Automated (Unit Test) | $(date -I) | $AUTHZ_TEST_STATUS | [Link]($AUTHZ_TEST_URL) |"

# ... and so on for all automated controls
````

-----

### 4\. Remediation Tracking: Plan of Action & Milestones (POA\&M)

When any audit (internal or external) identifies a gap, we don't just "log a bug." We create a formal **Plan of Action and Milestones (POA\&M)** to ensure the finding is tracked, prioritized, and remediated.

  * **POA\&M Structure (`POAM.ts`):**

<!-- end list -->

```typescript
// src/security/AuditFramework.ts

export type FindingStatus = 'Identified' | 'Remediation In Progress' | 'Completed' | 'Risk Accepted';

/**
 * Defines a formal plan for remediating an audit finding.
 */
export interface POAM {
  readonly findingId: string; // e.g., "INT-2025-Q3-001"
  readonly auditRecordId: string; // Links back to the source audit record
  readonly description: string; // What is the gap?
  readonly riskRating: 'Low' | 'Medium' | 'High' | 'Critical';
  readonly remediationPlan: string; // What are the concrete steps to fix it?
  readonly owner: string; // e.g., "SPARC-Code-Implementer" or a specific team
  targetDate: string; // ISO 8601 Date
  completionDate?: string;
  status: FindingStatus;
  verificationMethod: string; // How will we prove it's fixed?
}
```

-----

### 5\. Memory Bank Integration: The Audit Record

The results of all quarterly and external audits are stored as immutable artifacts in the Memory Bank, creating a complete history of our security assurance activities.

**Memory Bank Template: Audit Record (`audit-record.yml`)**

```yaml
#---
# File: memory/security-reviewer/audits/2025-Q3_InternalAudit_AccessControl.yml
#---
apiVersion: sparc.dev/v1
kind: MemoryEntry
metadata:
  domain: security-reviewer
  layer: assurance
  title: "2025 Q3 Internal Audit: Access Control"
  author: "SPARC-Security-Reviewer"
  timestamp: "2025-10-01T14:00:00Z"
spec:
  summary: "Results of the quarterly internal audit focused on access control mechanisms across all services, including API authorization, role definitions, and infrastructure access."
  type: "Audit Record"
  
  scope:
    - "All API endpoints"
    - "IAM roles and policies for production environment"
    - "User role definitions and permissions"

  findings:
    - findingId: "INT-2025-Q3-001"
      title: "No Review Process for Stale User Roles"
      riskRating: "Medium"
      poamId: "POAM-2025-005" # Link to the remediation tracking item
      
    - findingId: "INT-2025-Q3-002"
      title: "API key for external partner service lacks expiry"
      riskRating: "High"
      poamId: "POAM-2025-006"

  overallAssessment: "Controls are generally effective. Two medium/high risk findings were identified, and POA&Ms have been created. Automated controls for API authorization are performing as expected."
```

