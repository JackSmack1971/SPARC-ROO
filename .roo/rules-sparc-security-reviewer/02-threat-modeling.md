# Security Reviewer Rule: 02 - Systematic Threat Modeling Framework

**Purpose:** To establish a proactive, continuous, and systematic process for identifying and mitigating security threats throughout the system's lifecycle. Threat modeling is not an audit; it is a core design activity that embeds security into the SPARC methodology, ensuring we build security in, not bolt it on.

---

### 1. Philosophy: Security as a Design Primitive

Our security posture is defined during the **Specification** and **Architecture** phases, not after deployment. The **SPARC-Security-Reviewer** facilitates a collaborative process to answer four key questions:

1.  What are we building?
2.  What can go wrong?
3.  What are we going to do about it?
4.  Did we do a good job?

This process transforms security from a compliance checklist into a foundational component of system design, making it everyone's responsibility.

---

### 2. The SPARC Threat Modeling Process

We adopt a structured process that integrates directly into the SPARC workflow, primarily using the **STRIDE** methodology for threat identification and **DREAD** for risk rating.

#### Step 1: Decompose the System (Architecture Phase)
Before we can find threats, we must understand the system.
* **Input:** Architecture diagrams and specifications from the **SPARC-Architect**.
* **Activity:** The Security-Reviewer and Architect collaborate to create **Data Flow Diagrams (DFDs)**. DFDs visualize processes, data stores, external entities, and, most importantly, **trust boundaries**. A trust boundary is any point where data crosses from a less trusted entity to a more trusted one (e.g., public internet to our API Gateway).
* **Example DFD (using Mermaid JS syntax):**
    ```mermaid
    graph TD
        A[User's Browser] -->|HTTPS Request| B(API Gateway);
        subgraph "Trust Boundary: VPC"
            B -->|mTLS| C{User Service};
            C -->|SQL| D[(User Database)];
            C -->|AMQP| E((Notification Queue));
        end
    ```
    

#### Step 2: Identify Threats with STRIDE (Refinement Phase)
For each element in the DFD (processes, data flows, data stores), we systematically brainstorm threats using the STRIDE model.

| Category              | Threat Description                                      | Example in Our Stack                                           |
| --------------------- | ------------------------------------------------------- | -------------------------------------------------------------- |
| **S**poofing          | Illegitimately assuming another's identity.             | Forging a JWT; session hijacking; IP spoofing.                 |
| **T**ampering         | Unauthorized modification of data.                      | NoSQL injection; modifying JWT claims; man-in-the-middle attacks. |
| **R**epudiation       | Denying having performed an action.                     | Lack of audit logs for critical operations (e.g., deletion). |
| **I**nformation Disclosure | Exposure of information to unauthorized individuals.    | Leaking PII in error messages; unencrypted S3 buckets; timing attacks. |
| **D**enial of Service | Preventing legitimate users from accessing the system. | ReDoS attacks; resource exhaustion via complex queries; flooding an API. |
| **E**levation of Privilege | Gaining capabilities without proper authorization.      | Horizontal (user A accesses user B's data) or vertical (user becomes admin). |

#### Step 3: Rate Threats with DREAD (Refinement Phase)
Once threats are identified, we quantify their risk to prioritize mitigation efforts.

* **DREAD Model:**
    * **D**amage: How bad is the impact if the exploit succeeds? (1-10)
    * **R**eproducibility: How easy is it to reproduce the attack? (1-10)
    * **E**xploitability: What is needed to conduct the attack? (1-10)
    * **A**ffected Users: What percentage of users are impacted? (1-10)
    * **D**iscoverability: How easy is it to find the vulnerability? (1-10)
* **Risk Score Calculation:** `Risk = (D + R + E + A + D) / 5`. Scores above 7 are considered High/Critical.

---

### 3. The Threat Modeling Framework in Code

To formalize this process, we use a code-based framework to define and manage threats.

```typescript
// src/security/ThreatFramework.ts

export type StrideCategory = 'Spoofing' | 'Tampering' | 'Repudiation' | 'InformationDisclosure' | 'DenialOfService' | 'ElevationOfPrivilege';
export type ThreatStatus = 'Identified' | 'Mitigated' | 'Accepted' | 'FalsePositive';

export interface DreadRating {
  damage: number;          // 1-10
  reproducibility: number; // 1-10
  exploitability: number;  // 1-10
  affectedUsers: number;   // 1-10
  discoverability: number; // 1-10
}

export interface Threat {
  readonly id: string; // e.g., THREAT-USER-001
  readonly title: string;
  readonly description: string;
  readonly component: string; // e.g., "User Service API"
  readonly strideCategory: StrideCategory;
  readonly dread: DreadRating;
  readonly riskScore: number;
  mitigationPlan: string;
  mitigationOwner: 'SPARC-Architect' | 'SPARC-Code-Implementer' | 'SPARC-TDD-Engineer';
  status: ThreatStatus;
  validationTestId?: string; // Link to the test case that verifies the fix
}

export class RiskCalculator {
  public static calculate(dread: DreadRating): number {
    const total = dread.damage + dread.reproducibility + dread.exploitability + dread.affectedUsers + dread.discoverability;
    const score = parseFloat((total / 5).toFixed(2));
    if (score < 0 || score > 10) {
      throw new Error('Invalid DREAD scores provided; final score out of bounds.');
    }
    return score;
  }

  public static getRiskLevel(score: number): 'Low' | 'Medium' | 'High' | 'Critical' {
    if (score >= 8.5) return 'Critical';
    if (score >= 7.0) return 'High';
    if (score >= 4.0) return 'Medium';
    return 'Low';
  }
}
````

-----

### 4\. Memory Bank Integration: The Threat Ledger

The output of every threat modeling session is a formal, actionable artifact stored in the Memory Bank. This collection of artifacts forms our **Threat Ledger**.

**Documentation Triggers:**

  * A new feature with a public-facing component is designed.
  * A major architectural change is proposed (e.g., adding a new microservice or data store).
  * A new external system is integrated.

**Memory Bank Template: Threat Model (`threat-model.yml`)**

```yaml
#---
# File: memory/security-reviewer/threat-model-profile-upload-feature.yml
#---
apiVersion: sparc.dev/v1
kind: MemoryEntry
metadata:
  domain: security-reviewer
  layer: design
  title: "Threat Model for User Profile Picture Upload Feature"
  author: "SPARC-Security-Reviewer"
  timestamp: "2025-08-05T23:58:00Z"
spec:
  summary: "A systematic threat modeling session for the new feature allowing users to upload profile pictures. The feature involves a new API endpoint, an S3 bucket for storage, and an image processing worker."
  type: "Threat Model"
  
  dfd: |
    User -> API Gateway (uploads image)
    API Gateway -> Upload Service (validates and gets pre-signed URL)
    Upload Service -> S3 (generates pre-signed URL)
    User -> S3 (uploads file directly using URL)
    S3 -> Image Processing Worker (triggers on new object)
    Image Processing Worker -> S3 (writes resized images)

  threats:
    - id: "THREAT-UPLOAD-001"
      title: "Arbitrary File Upload Leading to RCE"
      component: "Upload Service"
      strideCategory: "Tampering"
      dread:
        damage: 9
        reproducibility: 7
        exploitability: 6
        affectedUsers: 10
        discoverability: 5
      riskScore: 7.4 # High
      mitigationPlan: "The Upload Service MUST validate the file's MIME type and extension against a strict allow-list (`.jpg`, `.png`, `.gif`). The image processing worker MUST re-process the image, which inherently validates it, and discard the original."
      mitigationOwner: "SPARC-Code-Implementer"
      status: "Identified"
      validationTestId: "SEC-TEST-012"

    - id: "THREAT-UPLOAD-002"
      title: "Denial of Service via 'Image Bombs'"
      component: "Image Processing Worker"
      strideCategory: "DenialOfService"
      dread:
        damage: 7
        reproducibility: 8
        exploitability: 7
        affectedUsers: 10
        discoverability: 6
      riskScore: 7.6 # High
      mitigationPlan: "Implement resource limits on the worker. The image processing library MUST be configured with safeguards against decompression bombs and excessively large image dimensions."
      mitigationOwner: "SPARC-DevOps-Engineer"
      status: "Identified"
      validationTestId: "PERF-TEST-004"
```