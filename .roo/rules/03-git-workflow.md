# Git Workflow Practices
## Version Control Excellence for SPARC Development

### SPARC-Aligned Git Philosophy

Version control in SPARC development serves multiple purposes beyond simple code storage:
- **Traceability**: Every commit traces back to a specific SPARC phase and requirement
- **Quality Gates**: Git workflows enforce SPARC quality standards before code integration
- **Collaboration**: Multiple modes and developers can work simultaneously without conflicts
- **Memory Bank Integration**: Commits preserve context and decision rationale
- **Security**: Sensitive information never enters version control through proper practices

### Branch Strategy and Structure

#### GitFlow Adapted for SPARC Methodology

```
main (production-ready code only)
├── develop (integration branch for completed SPARC cycles)
│   ├── feature/sparc-auth-system (complete SPARC cycle for authentication)
│   ├── feature/sparc-user-management (complete SPARC cycle for user features)
│   ├── feature/sparc-api-gateway (complete SPARC cycle for API layer)
│   └── hotfix/security-patch-auth (emergency fixes)
├── release/v1.2.0 (release preparation and testing)
└── support/v1.1.x (long-term support branches)
```

#### Branch Naming Conventions

**Feature Branches** (Complete SPARC Cycles):
```bash
# Pattern: feature/sparc-{domain}-{feature-name}
feature/sparc-auth-user-login
feature/sparc-payment-subscription-management
feature/sparc-analytics-user-behavior-tracking
feature/sparc-api-rate-limiting

# For individual SPARC phases (when working in isolation):
feature/sparc-auth-specification
feature/sparc-auth-architecture
feature/sparc-auth-implementation
```

**Hotfix Branches** (Urgent Production Fixes):
```bash
# Pattern: hotfix/{severity}-{brief-description}
hotfix/critical-auth-bypass
hotfix/security-input-validation
hotfix/performance-database-timeout
```

**Release Branches** (Release Preparation):
```bash
# Pattern: release/v{major}.{minor}.{patch}
release/v1.0.0
release/v1.1.0
release/v2.0.0-beta
```

### Commit Message Standards

#### SPARC-Enhanced Conventional Commits

Every commit message must clearly indicate:
1. **SPARC Phase**: Which phase this commit represents
2. **Component**: What part of the system is affected
3. **Intent**: What change is being made and why
4. **Impact**: How this relates to requirements and quality gates

#### Commit Message Format
```
<sparc-phase>(<scope>): <description>

<detailed explanation of changes>

SPARC-Context:
- Phase: [Specification|Pseudocode|Architecture|Refinement|Completion]
- Quality Gate: [Passed|In Progress|Failed]
- Requirements: [REQ-001, REQ-002]
- Memory Bank: [Updated productContext.md, systemPatterns.md]

Breaking Changes: [if any]
Security Impact: [if any]
Performance Impact: [if any]
```

#### Commit Message Examples

**Specification Phase Commits:**
```
spec(auth): define user authentication requirements

- Complete functional requirements for login, registration, and password reset
- Define security requirements including 2FA and session management
- Establish performance criteria: <200ms response time for auth requests
- Create user journey mapping for all authentication flows

SPARC-Context:
- Phase: Specification
- Quality Gate: Passed
- Requirements: REQ-001, REQ-002, REQ-003
- Memory Bank: Updated productContext.md with auth requirements

Co-authored-by: sparc-specification-writer
```

**Architecture Phase Commits:**
```
arch(auth): implement OAuth 2.0 + JWT architecture

- Design token-based authentication with refresh token rotation
- Plan Redis session store for horizontal scalability
- Define security boundaries between auth service and application services
- Document threat model for authentication attack vectors

SPARC-Context:
- Phase: Architecture
- Quality Gate: Passed
- Requirements: REQ-001, REQ-004
- Memory Bank: Updated decisionLog.md with OAuth choice rationale

Security Impact: Implements secure token-based authentication
Breaking Changes: Replaces session-based auth
```

**Implementation Phase Commits:**
```
impl(auth): add JWT token generation and validation

- Implement secure JWT signing with RS256 algorithm
- Add token expiration and refresh mechanisms
- Create middleware for route protection
- Maintain 350-line file size limit in auth-service.js

SPARC-Context:
- Phase: Refinement
- Quality Gate: In Progress
- Requirements: REQ-001, REQ-005
- Memory Bank: Updated systemPatterns.md with JWT patterns

Security Impact: Enables secure API authentication
Performance Impact: <50ms token validation overhead
```

**Testing Phase Commits:**
```
test(auth): comprehensive test suite for authentication flows

- Unit tests for token generation, validation, and expiration
- Integration tests for complete login/logout flows
- Security tests for common attack vectors (brute force, token manipulation)
- Performance tests validating <200ms response time requirement

SPARC-Context:
- Phase: Refinement
- Quality Gate: Passed
- Requirements: REQ-001, REQ-002, REQ-003
- Memory Bank: Updated systemPatterns.md with testing patterns

Coverage: 95% test coverage achieved
```

**Completion Phase Commits:**
```
deploy(auth): production deployment configuration

- Add Kubernetes deployment manifests for auth service
- Configure monitoring and alerting for authentication metrics
- Set up automated security scanning in CI/CD pipeline
- Document operational runbooks for auth service management

SPARC-Context:
- Phase: Completion
- Quality Gate: Passed
- Requirements: All auth requirements completed
- Memory Bank: Updated progress.md - auth system complete
```

### SPARC-Specific Git Hooks

#### Pre-Commit Hooks (Quality Gates)
```bash
#!/bin/bash
# .git/hooks/pre-commit

echo "🔍 SPARC Quality Gate: Pre-commit validation"

# 1. Enforce 500-line file limit
echo "Checking file size limits..."
large_files=$(find src/ -name "*.js" -o -name "*.ts" -o -name "*.py" | xargs wc -l | awk '$1 > 500 {print $2 " (" $1 " lines)"}')
if [ ! -z "$large_files" ]; then
    echo "❌ SPARC Violation: Files exceed 500-line limit:"
    echo "$large_files"
    echo "Please refactor these files to maintain SPARC modularity principles."
    exit 1
fi

# 2. Check for hardcoded secrets
echo "Scanning for hardcoded secrets..."
secrets_found=$(git diff --cached --name-only | xargs grep -l "api[_-]key\|secret\|password\|token" 2>/dev/null | grep -v ".env.example" | grep -v ".md")
if [ ! -z "$secrets_found" ]; then
    echo "❌ Security Violation: Potential secrets found in:"
    echo "$secrets_found"
    echo "Remove hardcoded secrets and use environment variables."
    exit 1
fi

# 3. Validate commit message format
commit_msg_file="$1"
if [ -f "$commit_msg_file" ]; then
    if ! grep -qE "^(spec|arch|impl|test|deploy|refactor|docs|fix)" "$commit_msg_file"; then
        echo "❌ Commit message must start with SPARC phase prefix:"
        echo "spec|arch|impl|test|deploy|refactor|docs|fix"
        exit 1
    fi
fi

# 4. Run tests for changed code
echo "Running tests for changed files..."
changed_files=$(git diff --cached --name-only | grep -E "\.(js|ts|py)$")
if [ ! -z "$changed_files" ]; then
    npm test -- --passWithNoTests --findRelatedTests $changed_files
    if [ $? -ne 0 ]; then
        echo "❌ Tests failed. Fix tests before committing."
        exit 1
    fi
fi

# 5. Check Memory Bank updates for architecture/design changes
arch_changes=$(git diff --cached --name-only | grep -E "(architecture|design|spec)" | wc -l)
memory_bank_changes=$(git diff --cached --name-only | grep "memory-bank/" | wc -l)

if [ "$arch_changes" -gt 0 ] && [ "$memory_bank_changes" -eq 0 ]; then
    echo "⚠️  Warning: Architectural changes detected but memory-bank/ not updated."
    echo "Consider updating memory-bank/decisionLog.md or memory-bank/systemPatterns.md"
    echo "Continue anyway? (y/N)"
    read -r response
    if [[ ! "$response" =~ ^[Yy]$ ]]; then
        exit 1
    fi
fi

echo "✅ All SPARC quality gates passed"
```

#### Pre-Push Hooks (Integration Quality Gates)
```bash
#!/bin/bash
# .git/hooks/pre-push

echo "🚀 SPARC Quality Gate: Pre-push validation"

# 1. Ensure all tests pass
echo "Running full test suite..."
npm test
if [ $? -ne 0 ]; then
    echo "❌ Full test suite must pass before pushing"
    exit 1
fi

# 2. Run security audit
echo "Running security audit..."
npm audit --audit-level=moderate
if [ $? -ne 0 ]; then
    echo "❌ Security vulnerabilities detected. Fix before pushing."
    exit 1
fi

# 3. Verify no merge conflicts remain
echo "Checking for merge conflict markers..."
conflict_files=$(git diff --name-only | xargs grep -l "<<<<<<< HEAD\|>>>>>>> \|=======" 2>/dev/null)
if [ ! -z "$conflict_files" ]; then
    echo "❌ Merge conflict markers found in:"
    echo "$conflict_files"
    exit 1
fi

# 4. Ensure Memory Bank is synchronized
echo "Validating Memory Bank consistency..."
if [ -f "memory-bank/progress.md" ]; then
    # Check if current branch work is documented in progress
    current_branch=$(git branch --show-current)
    if ! grep -q "$current_branch" memory-bank/progress.md; then
        echo "⚠️  Current branch not documented in memory-bank/progress.md"
        echo "Consider updating progress tracking before pushing."
    fi
fi

echo "✅ All integration quality gates passed"
```

### Collaborative Development Workflow

#### Multi-Mode Development Process

When multiple SPARC modes are working on the same feature:

```bash
# 1. Create feature branch from develop
git checkout develop
git pull origin develop
git checkout -b feature/sparc-user-profile

# 2. Specification Phase (sparc-specification-writer mode)
# Work on requirements and specifications
git add docs/specification.md memory-bank/productContext.md
git commit -m "spec(user): define user profile management requirements

Complete functional requirements for user profile CRUD operations
Define data validation rules and privacy controls
Establish API contract for profile service

SPARC-Context:
- Phase: Specification
- Quality Gate: Passed
- Requirements: REQ-015, REQ-016
- Memory Bank: Updated productContext.md"

# 3. Architecture Phase (sparc-architect mode)
git add docs/architecture.md memory-bank/decisionLog.md
git commit -m "arch(user): design microservice architecture for profiles

Separate profile service from user authentication service
Plan data synchronization between services
Design caching strategy for profile data access

SPARC-Context:
- Phase: Architecture
- Quality Gate: Passed
- Requirements: REQ-015
- Memory Bank: Updated decisionLog.md with service separation rationale"

# 4. Implementation Phase (multiple modes working in parallel)
# TDD Engineer creates tests first
git add tests/unit/user-profile/ tests/integration/user-profile/
git commit -m "test(user): comprehensive test suite for profile service

Unit tests for profile validation, updates, and privacy controls
Integration tests for profile API endpoints
Performance tests for profile data retrieval

SPARC-Context:
- Phase: Refinement
- Quality Gate: Passed
- Requirements: REQ-015, REQ-016"

# Code Implementer adds implementation
git add src/user-profile/
git commit -m "impl(user): implement user profile service

Add profile CRUD operations with validation
Implement privacy controls and data filtering
Maintain 400-line limit across all profile modules

SPARC-Context:
- Phase: Refinement
- Quality Gate: Passed
- Requirements: REQ-015, REQ-016
- Memory Bank: Updated systemPatterns.md with profile patterns"

# 5. Security Review and DevOps
# Security Reviewer validates implementation
git add docs/security/profile-security-audit.md
git commit -m "security(user): complete security audit of profile service

Validated input sanitization and authorization controls
Confirmed no PII leakage in error messages
Verified secure data storage and transmission

SPARC-Context:
- Phase: Refinement
- Quality Gate: Passed
- Security: No critical vulnerabilities found"

# DevOps Engineer adds deployment configuration
git add infrastructure/user-profile/ ci/user-profile-pipeline.yml
git commit -m "deploy(user): add deployment configuration for profile service

Kubernetes manifests for profile service deployment
CI/CD pipeline with security scanning and testing
Monitoring and alerting configuration

SPARC-Context:
- Phase: Completion
- Quality Gate: Passed
- Requirements: All profile requirements complete"
```

#### Merge Request/Pull Request Standards

Every merge request must include:

**1. SPARC Completion Checklist:**
```markdown
## SPARC Quality Gates Checklist

### Specification Phase
- [ ] Requirements documented in docs/specification.md
- [ ] Acceptance criteria defined and testable
- [ ] User scenarios documented
- [ ] Non-functional requirements specified

### Pseudocode Phase
- [ ] Algorithm design documented in docs/pseudocode.md
- [ ] Function signatures and interfaces defined
- [ ] Complexity analysis completed

### Architecture Phase
- [ ] System design documented in docs/architecture.md
- [ ] Technology choices justified in memory-bank/decisionLog.md
- [ ] Integration points defined
- [ ] Security architecture reviewed

### Refinement Phase
- [ ] All code follows 500-line file limit
- [ ] Test coverage >90% for new code
- [ ] Security review completed with no critical issues
- [ ] Performance requirements validated
- [ ] Code review completed

### Completion Phase
- [ ] Deployment configuration ready
- [ ] Documentation updated
- [ ] Monitoring and alerting configured
- [ ] Memory Bank updated with lessons learned
```

**2. Impact Analysis:**
```markdown
## Change Impact Analysis

### Requirements Addressed
- REQ-015: User Profile Management
- REQ-016: Privacy Controls

### Files Modified
- src/user-profile/ (new module, 3 files, avg 350 lines each)
- tests/unit/user-profile/ (comprehensive test coverage)
- docs/architecture.md (service architecture updates)
- memory-bank/decisionLog.md (architectural decisions)

### Breaking Changes
None

### Security Implications
- Added new API endpoints with proper authorization
- Implemented PII protection controls
- No sensitive data in logs or error messages

### Performance Impact
- Profile data caching reduces database load by ~60%
- New endpoints tested under load (500 RPS, <100ms response)

### Memory Bank Updates
- systemPatterns.md: Added profile data validation patterns
- decisionLog.md: Documented microservice separation decision
- progress.md: Marked user profile feature as complete
```

**3. Testing Evidence:**
```markdown
## Testing Validation

### Test Coverage
- Unit Tests: 96% coverage
- Integration Tests: All API endpoints covered
- Security Tests: OWASP Top 10 validated
- Performance Tests: All requirements met

### Quality Metrics
- ESLint: 0 errors, 0 warnings
- Security Scan: 0 critical, 0 high severity issues
- Bundle Size: +15KB (within acceptable limits)

### Manual Testing
- [ ] User profile creation flow
- [ ] Profile update with validation
- [ ] Privacy controls enforcement
- [ ] Error handling scenarios
```

### Release Management

#### SPARC-Aligned Release Process

**1. Release Branch Creation:**
```bash
# Create release branch from develop
git checkout develop
git pull origin develop
git checkout -b release/v1.2.0

# Update version numbers and changelog
git add package.json CHANGELOG.md
git commit -m "release: prepare v1.2.0

- User profile management feature complete
- Authentication system security enhancements
- Performance optimizations for API endpoints

SPARC-Context:
- Phase: Completion
- Quality Gate: Ready for release
- Features: Complete SPARC cycles for 3 major features"
```

**2. Release Testing and Validation:**
```bash
# Run comprehensive test suite
npm run test:full

# Run security audit
npm audit

# Performance benchmarking
npm run benchmark

# Update Memory Bank with release notes
git add memory-bank/progress.md
git commit -m "docs: update Memory Bank for v1.2.0 release

- Document completed features and architectural decisions
- Update system patterns with release learnings
- Mark milestone completion in progress tracking"
```

**3. Release Deployment:**
```bash
# Merge to main for production deployment
git checkout main
git merge release/v1.2.0 --no-ff
git tag -a v1.2.0 -m "Release v1.2.0: User Profile Management

Complete SPARC implementation including:
- Comprehensive user profile management
- Enhanced security controls
- Performance optimizations
- Full test coverage and documentation"

# Deploy to production
git push origin main --tags
```

### Emergency Hotfix Workflow

#### Critical Issue Response Process

**1. Immediate Response:**
```bash
# Create hotfix branch from main (production)
git checkout main
git pull origin main
git checkout -b hotfix/critical-auth-bypass

# Implement minimal fix following SPARC security principles
git add src/auth/auth-middleware.js
git commit -m "fix(auth): patch critical authentication bypass vulnerability

- Add missing authorization check in protected route middleware
- Implement input validation for user context
- Maintain existing functionality while closing security gap

SPARC-Context:
- Phase: Emergency Refinement
- Quality Gate: Security patch verified
- Security Impact: Closes critical authentication bypass
- Requirements: Maintains all existing auth requirements

CVE: Internal-2024-001
Severity: Critical"
```

**2. Hotfix Validation:**
```bash
# Run targeted security tests
npm run test:security

# Run regression tests
npm run test:regression

# Manual security validation
npm run security:validate-auth

# Update Memory Bank with incident learning
git add memory-bank/decisionLog.md
git commit -m "security: document auth bypass incident and resolution

- Record root cause analysis of vulnerability
- Document prevention measures for similar issues
- Update security review checklist based on findings

SPARC-Context:
- Phase: Completion
- Security: Incident response documented
- Memory Bank: Updated with security patterns"
```

**3. Emergency Deployment:**
```bash
# Deploy hotfix immediately
git checkout main
git merge hotfix/critical-auth-bypass --no-ff
git tag -a v1.1.1 -m "Hotfix v1.1.1: Critical security patch"
git push origin main --tags

# Backport to develop
git checkout develop
git merge hotfix/critical-auth-bypass --no-ff
git push origin develop

# Clean up hotfix branch
git branch -d hotfix/critical-auth-bypass
git push origin --delete hotfix/critical-auth-bypass
```

### Memory Bank Integration with Git

#### Automatic Memory Bank Updates

**Git Integration Script (.git/hooks/post-commit):**
```bash
#!/bin/bash
# Automatically update Memory Bank based on commit content

commit_message=$(git log -1 --pretty=%B)
commit_hash=$(git rev-parse HEAD)
commit_date=$(git log -1 --pretty=%cd --date=iso)

# Extract SPARC context from commit message
sparc_phase=$(echo "$commit_message" | grep -o "Phase: [^-]*" | cut -d' ' -f2)
requirements=$(echo "$commit_message" | grep -o "Requirements: [^-]*" | cut -d' ' -f2-)

# Update progress.md with commit information
if [ ! -z "$sparc_phase" ]; then
    echo "## Commit $commit_hash - $commit_date" >> memory-bank/progress.md
    echo "**Phase:** $sparc_phase" >> memory-bank/progress.md
    echo "**Requirements:** $requirements" >> memory-bank/progress.md
    echo "**Message:** $(echo "$commit_message" | head -1)" >> memory-bank/progress.md
    echo "" >> memory-bank/progress.md
fi

# Auto-commit Memory Bank updates (if changes detected)
if [ -n "$(git status --porcelain memory-bank/)" ]; then
    git add memory-bank/
    git commit -m "auto: update Memory Bank from commit $commit_hash"
fi
```

This Git workflow ensures that every aspect of SPARC development maintains traceability, quality, and collaborative excellence while preserving the project's architectural integrity and security standards.