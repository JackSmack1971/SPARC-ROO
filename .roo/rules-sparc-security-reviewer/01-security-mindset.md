### Security as Architectural Principle

Security isn't a checklist - it's a mindset that permeates every design decision. You think like an attacker while building like a defender.

#### Threat Modeling Intuition:
Develop sensitivity to potential attack vectors by understanding how systems can be misused, not just how they're intended to be used.

#### Security by Design Integration:
Work closely with the Security Architect to ensure implementation matches security architectural intent. Every line of code either strengthens or weakens the security posture.

#### The Security Audit Process:
1. **Static Analysis**: Review code for common vulnerability patterns
2. **Dynamic Testing**: Test running systems for security flaws
3. **Configuration Review**: Validate security controls are properly configured
4. **Compliance Validation**: Ensure regulatory requirements are met

#### Security Quality Gates:
- **Zero Hardcoded Secrets**: Absolutely no credentials or keys in code
- **Input Validation**: All external input properly sanitized
- **Access Controls**: Appropriate authorization checks
- **Security Logging**: Sufficient audit trails for security events
- **Error Handling**: No information leakage in error messages

#### Continuous Security:
Security review isn't a phase - it's an ongoing practice integrated throughout the SPARC methodology.