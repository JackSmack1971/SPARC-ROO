# DevOps Engineer Rule: 04 - Security Operations (SecOps) Framework

**Purpose:** To define the processes and automation for detecting, responding to, and recovering from security incidents. This framework integrates security into our daily operations, ensuring the resilience and integrity of our production environment by assuming breaches are not a matter of *if*, but *when*.

---

### 1. Philosophy: Automate, Detect, Respond

Our SecOps posture is built on a foundation of proactive automation and practiced response. We operate under three core principles:

1.  **Automate Detection:** We use machines to find anomalous and malicious behavior by analyzing our rich telemetry data. Human operators investigate high-confidence signals, not raw logs.
2.  **Practice Response:** Our Incident Response (IR) plan is not a dusty document. It is a living playbook that is practiced regularly through "Game Day" simulations to build muscle memory.
3.  **Security is a Team Sport:** While the DevOps and Security roles lead SecOps, all engineers are first responders and are expected to participate in incident analysis and remediation.

---

### 2. Core SecOps Components

Our SecOps capability is built upon a stack of integrated, automated tools and processes.

#### 2.1. Centralized Telemetry and SIEM
* **Standard:** All telemetry—structured application logs, AWS CloudTrail logs, VPC Flow Logs, API Gateway access logs, and security tool outputs—**MUST** be streamed to our central Security Information and Event Management (SIEM) platform (e.g., Elastic Security, Panther).
* **Benefit:** This provides a single pane of glass for correlating events across our entire stack, allowing us to trace an attack from a network anomaly to an application error.

#### 2.2. Automated Threat Detection
* **Standard:** We maintain a version-controlled library of detection rules within our SIEM. These rules codify adversary tactics, techniques, and procedures (TTPs), primarily based on the **MITRE ATT&CK® framework**.
* **Example Detection-as-Code Rule:**

```yaml
# Rule ID: DETECT-IAM-001
# Title: Static AWS Credentials Committed to Public Repository
# MITRE Tactic: TA0001 (Initial Access)

# This pseudo-code represents a rule in our SIEM.
- name: "Leaked AWS Access Key Detected via GitHub Scan"
  # This rule triggers on an alert from our GitHub repo scanner (e.g., GitGuardian)
  source: "github_alerts"
  filter: "alert.type == 'aws_access_key'"
  
  # Correlate the key with our IAM inventory
  enrichment:
    - action: "lookup_iam_user"
      field: "alert.key_id"

  # The final alert payload
  alert:
    severity: "Critical"
    summary: "Static AWS key for user `${iam_user.name}` was detected in a public repository."
    # The playbook ID links directly to the runbook for this specific alert
    playbook_id: "IRP-001" 
````

#### 2.3. Secrets Management

  * **Standard:** All application secrets (database passwords, API keys, certificates) **MUST** be managed by a central secret vault (e.g., HashiCorp Vault, AWS Secrets Manager).
  * **Enforcement:**
      * Applications retrieve secrets at runtime using IAM Roles. Secrets are **NEVER** stored in configuration files, environment variables, or source code.
      * Secrets **MUST** be configured for automatic rotation with a maximum lifetime of 90 days.

#### 2.4. Vulnerability Management

  * **In Pipeline:** The `security-scan` job in our CI pipeline fails the build if new high or critical vulnerabilities are detected in our code, dependencies, or base container images.
  * **In Production:** We run scheduled, continuous scans of our production environment to detect vulnerabilities in running containers and infrastructure.
  * **Remediation SLA:** Vulnerabilities are tracked as formal `POA&M`s with strict SLAs:
      * **Critical:** 7 days
      * **High:** 30 days

-----

### 3\. The Incident Response (IR) Plan

When a critical alert fires, we follow a formal, phased IR plan.

1.  **Preparation:** (Ongoing) Practice the IR plan via quarterly Game Day simulations. Ensure all engineers have the necessary access and training.
2.  **Detection & Analysis:** (T+0h) The on-call engineer receives an alert from the SIEM. They are responsible for validating the alert within 15 minutes and declaring an "incident" if it's a true positive.
3.  **Containment:** (T+1h) The immediate goal is to stop the bleeding. Isolate the affected component from the network (e.g., by modifying security group rules), rotate compromised credentials, and block attacker IP addresses.
4.  **Eradication & Recovery:** (T+2-12h) Identify and remove the root cause (e.g., patch the vulnerability, remove malicious files). Restore system functionality from a known-good state (e.g., redeploying an immutable image).
5.  **Post-Incident Activity:** (T+1-3 days) This is the most crucial phase. A blameless post-mortem is conducted to understand the root cause and improve our defenses.

-----

### 4\. Memory Bank Integration: Post-Mortem Records

The lessons learned from every security incident are our most valuable asset for building a more resilient system. These are captured in formal post-mortem records.

**Memory Bank Template: Post-Mortem Record (`post-mortem-record.yml`)**

```yaml
#---
# File: memory/devops-engineer/post-mortems/2025-08-05_LeakedAWSCredential.yml
#---
apiVersion: sparc.dev/v1
kind: MemoryEntry
metadata:
  domain: devops-engineer
  layer: incident-response
  title: "Post-Mortem for Incident 2025-08-05: Leaked AWS Credential"
  author: "SPARC-Security-Reviewer"
  timestamp: "2025-08-06T00:45:00Z"
spec:
  summary: "Post-mortem analysis of a security incident where a static IAM user access key was accidentally committed to a public GitHub repository and used by an external actor."
  type: "Post-Mortem Record"
  
  timeline:
    - "2025-08-05 23:15 UTC: Developer accidentally commits static AWS key to public fork."
    - "2025-08-05 23:22 UTC: Automated GitHub scanner detects the key and triggers a CRITICAL SIEM alert (DETECT-IAM-001)."
    - "2025-08-05 23:25 UTC: On-call DevOps engineer acknowledges the alert and declares an incident."
    - "2025-08-05 23:30 UTC: (Containment) The compromised IAM user's credentials are immediately deactivated."
    - "2025-08-05 23:45 UTC: (Eradication) The public repository is deleted. CloudTrail logs are analyzed; no malicious activity beyond reconnaissance calls was detected."
    - "2025-08-06 00:00 UTC: Incident resolved."
  
  rootCauseAnalysis: "The root cause was twofold: 1) A static, long-lived IAM user key was in use, which is against policy. 2) The developer's local pre-commit hooks, which should have caught the secret, were not installed correctly."

  actionItems:
    - poamId: "POAM-2025-007"
      description: "Audit all IAM users and disable/remove all static access keys. Enforce the use of IAM Roles for all EC2/Lambda/GitHub Actions access."
      owner: "SPARC-DevOps-Engineer"
      targetDate: "2025-08-13"

    - poamId: "POAM-2025-008"
      description: "Integrate secret scanning (trufflehog) directly into the CI pipeline's PR check, making it a server-side gate rather than relying on client-side hooks."
      owner: "SPARC-DevOps-Engineer"
      targetDate: "2025-08-20"
```