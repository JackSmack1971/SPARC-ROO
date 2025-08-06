1798.110, 1798.115)
    request_type ccpa_request_enum NOT NULL,
    -- 'know_categories', 'know_specific', 'delete', 'opt_out_sale', 'opt_out_sharing', 'correct', 'limit_sensitive'
    
    -- Request details
    requested_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    request_method VARCHAR(100) NOT NULL, -- 'web_form', 'email', 'phone', 'mail'
    time_period_requested INTERVAL, -- For data requests
    specific_categories TEXT[], -- Requested PI categories
    
    -- Processing timeline (CCPA 1798.130)
    acknowledgment_due TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '10 days',
    response_due TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '45 days',
    acknowledged_at TIMESTAMPTZ,
    
    -- Status tracking
    status ccpa_status_enum NOT NULL DEFAULT 'pending',
    -- 'pending', 'acknowledged', 'in_progress', 'extended', 'completed', 'denied'
    denial_reason TEXT,
    extension_reason TEXT,
    extended_due_date TIMESTAMPTZ,
    
    -- Response delivery
    response_method VARCHAR(100),
    response_delivered_at TIMESTAMPTZ,
    response_format VARCHAR(50), -- 'json', 'csv', 'pdf', 'mail'
    
    -- Compliance tracking
    processed_by UUID,
    privacy_officer_reviewed BOOLEAN DEFAULT FALSE,
    compliance_notes TEXT,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Personal Information categories as defined by CCPA
CREATE TABLE ccpa_pi_categories (
    category_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category_code VARCHAR(50) UNIQUE NOT NULL,
    category_name VARCHAR(200) NOT NULL,
    category_description TEXT NOT NULL,
    
    -- CCPA classification
    is_sensitive_pi BOOLEAN NOT NULL DEFAULT FALSE,
    statutory_reference VARCHAR(100), -- e.g., "1798.140(ae)(1)(A)"
    
    -- Business context
    collection_purposes TEXT[] NOT NULL,
    retention_period INTERVAL,
    third_party_sharing BOOLEAN DEFAULT FALSE,
    
    effective_from TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    superseded_at TIMESTAMPTZ
);

-- Map database columns to CCPA PI categories
CREATE TABLE ccpa_data_mapping (
    mapping_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Database location
    schema_name VARCHAR(100) NOT NULL,
    table_name VARCHAR(100) NOT NULL,
    column_name VARCHAR(100) NOT NULL,
    
    -- CCPA classification
    pi_category_id UUID NOT NULL REFERENCES ccpa_pi_categories(category_id),
    collection_purpose TEXT NOT NULL,
    source_of_collection VARCHAR(200) NOT NULL,
    
    -- Sharing and selling tracking
    sold_to_third_parties BOOLEAN NOT NULL DEFAULT FALSE,
    shared_with_third_parties BOOLEAN NOT NULL DEFAULT FALSE,
    third_party_categories TEXT[],
    
    -- Consumer rights applicability
    subject_to_deletion BOOLEAN NOT NULL DEFAULT TRUE,
    subject_to_correction BOOLEAN NOT NULL DEFAULT TRUE,
    deletion_exemptions ccpa_exemption_enum[],
    
    -- Governance
    mapped_by VARCHAR(200) NOT NULL,
    approved_by VARCHAR(200),
    approved_at TIMESTAMPTZ,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    effective_from TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    CONSTRAINT ccpa_mapping_unique UNIQUE (schema_name, table_name, column_name, effective_from)
);
```

### International Privacy Standards

#### Multi-Jurisdictional Compliance Framework

**Universal Privacy Controls**:
```sql
-- Global privacy framework supporting multiple regulations
CREATE TABLE privacy_jurisdictions (
    jurisdiction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Jurisdiction details
    jurisdiction_code VARCHAR(10) UNIQUE NOT NULL, -- 'GDPR', 'CCPA', 'PIPEDA', 'LGPD'
    jurisdiction_name VARCHAR(200) NOT NULL,
    region VARCHAR(100) NOT NULL,
    
    -- Legal framework
    primary_regulation VARCHAR(200) NOT NULL,
    effective_date DATE NOT NULL,
    major_revision_date DATE,
    
    -- Key requirements
    consent_required BOOLEAN NOT NULL,
    explicit_consent_for_sensitive BOOLEAN NOT NULL,
    right_to_erasure BOOLEAN NOT NULL,
    right_to_portability BOOLEAN NOT NULL,
    right_to_rectification BOOLEAN NOT NULL,
    data_breach_notification_hours INTEGER,
    
    -- Response timeframes
    max_response_days INTEGER NOT NULL,
    acknowledgment_days INTEGER,
    
    -- Penalties
    max_fine_percentage DECIMAL(5,2),
    max_fine_amount BIGINT,
    currency_code VARCHAR(3),
    
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Subject rights matrix across jurisdictions
CREATE TABLE subject_rights_matrix (
    right_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES privacy_jurisdictions(jurisdiction_id),
    
    -- Right definition
    right_type subject_right_enum NOT NULL,
    -- 'access', 'rectification', 'erasure', 'portability', 'restriction', 'objection', 'opt_out'
    right_name VARCHAR(200) NOT NULL,
    statutory_basis VARCHAR(200),
    
    -- Implementation requirements
    identity_verification_required BOOLEAN NOT NULL,
    fee_permitted BOOLEAN NOT NULL DEFAULT FALSE,
    maximum_fee_amount DECIMAL(10,2),
    
    -- Exemptions and limitations
    exemptions TEXT[],
    limitations TEXT[],
    
    -- Technical requirements
    response_format_options TEXT[] NOT NULL,
    delivery_method_options TEXT[] NOT NULL,
    
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    CONSTRAINT rights_matrix_unique UNIQUE (jurisdiction_id, right_type)
);
```

### Data Minimization and Purpose Limitation

#### Collection Limitation Patterns

**Purpose-Bound Data Collection**:
```sql
-- Enforce purpose limitation (GDPR Article 5.1.b)
CREATE TABLE data_collection_purposes (
    purpose_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Purpose definition
    purpose_name VARCHAR(200) UNIQUE NOT NULL,
    purpose_description TEXT NOT NULL,
    business_justification TEXT NOT NULL,
    
    -- Legal basis
    legal_basis gdpr_legal_basis_enum NOT NULL,
    legitimate_interests_assessment TEXT,
    
    -- Data minimization
    required_data_categories TEXT[] NOT NULL,
    prohibited_data_categories TEXT[] DEFAULT '{}',
    retention_period INTERVAL NOT NULL,
    
    -- Geographic scope
    geographic_restrictions TEXT[],
    cross_border_transfers BOOLEAN DEFAULT FALSE,
    adequacy_decision_countries TEXT[],
    
    -- Consent requirements
    requires_explicit_consent BOOLEAN NOT NULL,
    consent_language TEXT,
    
    -- Approval workflow
    approved_by VARCHAR(200),
    approved_at TIMESTAMPTZ,
    legal_review_completed BOOLEAN DEFAULT FALSE,
    dpo_approved BOOLEAN DEFAULT FALSE,
    
    -- Lifecycle
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    deactivated_at TIMESTAMPTZ,
    deactivation_reason TEXT,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Link data processing to specific purposes
CREATE TABLE data_processing_activities (
    activity_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Processing identification
    activity_name VARCHAR(200) NOT NULL,
    system_name VARCHAR(100) NOT NULL,
    controller_name VARCHAR(200) NOT NULL,
    processor_name VARCHAR(200),
    
    -- Purpose linkage
    purpose_id UUID NOT NULL REFERENCES data_collection_purposes(purpose_id),
    data_categories_processed TEXT[] NOT NULL,
    
    -- Technical details
    processing_methods TEXT[] NOT NULL, -- ['automated', 'manual', 'ml_inference']
    storage_locations TEXT[] NOT NULL,
    retention_schedule INTERVAL NOT NULL,
    
    -- Security measures
    technical_measures TEXT[] NOT NULL,
    organizational_measures TEXT[] NOT NULL,
    encryption_in_transit BOOLEAN NOT NULL,
    encryption_at_rest BOOLEAN NOT NULL,
    
    -- Cross-border transfers
    international_transfers BOOLEAN DEFAULT FALSE,
    transfer_mechanisms TEXT[], -- ['adequacy_decision', 'scc', 'bcr']
    third_countries TEXT[],
    
    -- Risk assessment
    risk_level risk_level_enum NOT NULL, -- 'low', 'medium', 'high'
    dpia_required BOOLEAN DEFAULT FALSE,
    dpia_completed_at TIMESTAMPTZ,
    
    -- Lifecycle
    started_at TIMESTAMPTZ NOT NULL,
    scheduled_end_date TIMESTAMPTZ,
    actual_end_date TIMESTAMPTZ,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Retention and Disposal Architecture

#### Automated Retention Management

**Retention Policy Engine**:
```sql
-- Centralized retention policy management
CREATE TABLE data_retention_policies (
    policy_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Policy identification
    policy_name VARCHAR(200) UNIQUE NOT NULL,
    policy_description TEXT NOT NULL,
    jurisdiction_id UUID REFERENCES privacy_jurisdictions(jurisdiction_id),
    
    -- Retention rules
    base_retention_period INTERVAL NOT NULL,
    retention_trigger retention_trigger_enum NOT NULL,
    -- 'creation_date', 'last_interaction', 'account_closure', 'contract_end', 'consent_withdrawal'
    
    -- Extension conditions
    legal_hold_extension BOOLEAN DEFAULT FALSE,
    litigation_hold_extension BOOLEAN DEFAULT FALSE,
    regulatory_investigation_extension BOOLEAN DEFAULT FALSE,
    
    -- Disposal method
    disposal_method disposal_method_enum NOT NULL,
    -- 'hard_delete', 'anonymization', 'pseudonymization', 'archival'
    
    -- Automation settings
    automated_disposal BOOLEAN DEFAULT TRUE,
    warning_period INTERVAL DEFAULT INTERVAL '30 days',
    batch_processing BOOLEAN DEFAULT TRUE,
    
    -- Approval requirements
    requires_legal_approval BOOLEAN DEFAULT FALSE,
    requires_dpo_approval BOOLEAN DEFAULT FALSE,
    
    -- Governance
    created_by VARCHAR(200) NOT NULL,
    approved_by VARCHAR(200),
    approved_at TIMESTAMPTZ,
    
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    effective_from TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    superseded_at TIMESTAMPTZ,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Apply retention policies to specific data elements
CREATE TABLE data_retention_schedule (
    schedule_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Data location
    schema_name VARCHAR(100) NOT NULL,
    table_name VARCHAR(100) NOT NULL,
    record_identifier_column VARCHAR(100) NOT NULL,
    
    -- Policy application
    retention_policy_id UUID NOT NULL REFERENCES data_retention_policies(policy_id),
    custom_retention_period INTERVAL, -- Override policy default if needed
    
    -- Trigger configuration
    trigger_date_column VARCHAR(100) NOT NULL,
    trigger_calculation_method TEXT,
    
    -- Dependencies
    dependent_tables TEXT[], -- Tables that must be processed first
    cascading_deletions BOOLEAN DEFAULT TRUE,
    
    -- Processing controls
    processing_priority INTEGER DEFAULT 100,
    batch_size INTEGER DEFAULT 1000,
    max_processing_time INTERVAL DEFAULT INTERVAL '2 hours',
    
    -- Status tracking
    last_processed_at TIMESTAMPTZ,
    next_processing_due TIMESTAMPTZ,
    processing_errors TEXT[],
    
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    CONSTRAINT retention_schedule_unique UNIQUE (schema_name, table_name, retention_policy_id)
);

-- Automated retention processing log
CREATE TABLE retention_processing_log (
    log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Processing batch identification
    batch_id UUID NOT NULL,
    schedule_id UUID NOT NULL REFERENCES data_retention_schedule(schedule_id),
    
    -- Processing details
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    status processing_status_enum NOT NULL DEFAULT 'in_progress',
    -- 'in_progress', 'completed', 'failed', 'partial'
    
    -- Results
    records_evaluated INTEGER NOT NULL DEFAULT 0,
    records_disposed INTEGER NOT NULL DEFAULT 0,
    records_extended INTEGER NOT NULL DEFAULT 0,
    records_errors INTEGER NOT NULL DEFAULT 0,
    
    -- Error tracking
    error_details JSONB,
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    
    -- Compliance proof
    disposal_certificate_hash VARCHAR(255),
    verification_completed BOOLEAN DEFAULT FALSE,
    
    processed_by VARCHAR(200) NOT NULL,
    processing_system VARCHAR(100) NOT NULL
);
```

### Privacy Impact Assessment Integration

#### DPIA Workflow Management

**Data Protection Impact Assessment Framework**:
```sql
-- DPIA management for high-risk processing
CREATE TABLE data_protection_impact_assessments (
    dpia_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Assessment identification
    assessment_name VARCHAR(200) NOT NULL,
    processing_activity_id UUID REFERENCES data_processing_activities(activity_id),
    
    -- Risk triggers (GDPR Article 35.3)
    systematic_monitoring BOOLEAN NOT NULL,
    large_scale_sensitive_data BOOLEAN NOT NULL,
    public_area_monitoring BOOLEAN NOT NULL,
    automated_decision_making BOOLEAN NOT NULL,
    vulnerable_data_subjects BOOLEAN NOT NULL,
    innovative_technology BOOLEAN NOT NULL,
    transfer_outside_eu BOOLEAN NOT NULL,
    
    -- Assessment details
    processing_description TEXT NOT NULL,
    necessity_assessment TEXT NOT NULL,
    proportionality_assessment TEXT NOT NULL,
    
    -- Risk analysis
    identified_risks JSONB NOT NULL,
    risk_likelihood risk_likelihood_enum NOT NULL,
    risk_severity risk_severity_enum NOT NULL,
    overall_risk_score INTEGER CHECK (overall_risk_score BETWEEN 1 AND 25),
    
    -- Mitigation measures
    technical_measures TEXT[] NOT NULL,
    organizational_measures TEXT[] NOT NULL,
    safeguards_implemented TEXT[] NOT NULL,
    residual_risk_level risk_level_enum NOT NULL,
    
    -- Consultation requirements
    dpo_consulted BOOLEAN NOT NULL DEFAULT FALSE,
    dpo_consultation_date TIMESTAMPTZ,
    dpo_recommendations TEXT,
    
    supervisory_authority_consulted BOOLEAN DEFAULT FALSE,
    authority_consultation_date TIMESTAMPTZ,
    authority_opinion TEXT,
    
    -- Stakeholder input
    data_subjects_consulted BOOLEAN DEFAULT FALSE,
    consultation_method VARCHAR(200),
    stakeholder_feedback TEXT,
    
    -- Review and approval
    completed_by VARCHAR(200) NOT NULL,
    completed_at TIMESTAMPTZ,
    approved_by VARCHAR(200),
    approved_at TIMESTAMPTZ,
    
    -- Lifecycle management
    review_required_by TIMESTAMPTZ,
    last_reviewed_at TIMESTAMPTZ,
    next_review_due TIMESTAMPTZ,
    
    status dpia_status_enum NOT NULL DEFAULT 'draft',
    -- 'draft', 'in_review', 'authority_consultation', 'approved', 'rejected', 'requires_revision'
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Privacy Monitoring and Compliance Reporting

#### Automated Compliance Monitoring

**Privacy Compliance Dashboard**:
```sql
-- Real-time privacy compliance monitoring
CREATE TABLE privacy_compliance_metrics (
    metric_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Metric identification
    metric_name VARCHAR(200) NOT NULL,
    metric_category compliance_category_enum NOT NULL,
    -- 'consent_management', 'data_quality', 'retention_compliance', 'response_times', 'security_measures'
    
    -- Measurement details
    measurement_date TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metric_value DECIMAL(10,4) NOT NULL,
    metric_unit VARCHAR(50) NOT NULL,
    target_value DECIMAL(10,4) NOT NULL,
    
    -- Context
    jurisdiction_id UUID REFERENCES privacy_jurisdictions(jurisdiction_id),
    processing_activity_id UUID REFERENCES data_processing_activities(activity_id),
    data_source VARCHAR(200) NOT NULL,
    
    -- Status
    compliance_status compliance_status_enum NOT NULL,
    -- 'compliant', 'non_compliant', 'at_risk', 'unknown'
    
    -- Alerting
    alert_threshold DECIMAL(10,4),
    alert_triggered BOOLEAN DEFAULT FALSE,
    alert_escalated BOOLEAN DEFAULT FALSE,
    
    -- Remediation
    action_required TEXT,
    remediation_due_date TIMESTAMPTZ,
    remediation_completed BOOLEAN DEFAULT FALSE,
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Privacy audit trail for regulatory reporting
CREATE TABLE privacy_audit_events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Event classification
    event_type privacy_event_enum NOT NULL,
    -- 'consent_given', 'consent_withdrawn', 'data_access', 'data_correction', 'data_erasure', 'breach_detected'
    event_category VARCHAR(100) NOT NULL,
    
    -- Subject and object
    data_subject_id UUID,
    data_subject_type VARCHAR(100),
    affected_data_categories TEXT[] NOT NULL,
    
    -- Event details
    event_description TEXT NOT NULL,
    event_timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    system_name VARCHAR(100) NOT NULL,
    user_agent TEXT,
    ip_address INET,
    
    -- Legal context
    legal_basis gdpr_legal_basis_enum,
    processing_purpose VARCHAR(200),
    jurisdiction_codes TEXT[] NOT NULL,
    
    -- Technical details
    processing_method VARCHAR(100),
    data_location VARCHAR(200),
    encryption_used BOOLEAN,
    
    -- Compliance tracking
    regulatory_notification_required BOOLEAN DEFAULT FALSE,
    notification_sent BOOLEAN DEFAULT FALSE,
    notification_sent_at TIMESTAMPTZ,
    
    -- Retention for audit
    retention_until TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '10 years',
    
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Implementation Checklist

#### Pre-Implementation Privacy Review

✅ **Legal Framework Compliance**
- [ ] All applicable privacy regulations identified
- [ ] Legal basis established for all data processing
- [ ] Data minimization principles applied
- [ ] Purpose limitation enforced
- [ ] Consent mechanisms implemented where required

✅ **Technical Privacy Controls**
- [ ] Row-level security policies implemented
- [ ] Data classification completed
- [ ] Encryption at rest and in transit configured
- [ ] Anonymization/pseudonymization capabilities built
- [ ] Automated retention and disposal systems operational

✅ **Subject Rights Implementation**
- [ ] Right of access (data portability) implemented
- [ ] Right to rectification (correction) implemented
- [ ] Right to erasure (deletion) implemented
- [ ] Right to restriction of processing implemented
- [ ] Right to object to processing implemented
- [ ] Right to opt-out of sale/sharing (CCPA) implemented

✅ **Governance and Monitoring**
- [ ] Privacy impact assessments completed for high-risk processing
- [ ] Data protection officer involvement documented
- [ ] Cross-border transfer mechanisms implemented
- [ ] Breach detection and notification systems operational
- [ ] Regular compliance monitoring and reporting established

✅ **Documentation and Training**
- [ ] Privacy notices updated and accessible
- [ ] Data processing records maintained (GDPR Article 30)
- [ ] Staff training on privacy requirements completed
- [ ] Vendor agreements include privacy clauses
- [ ] Incident response procedures include privacy breach protocols

This comprehensive privacy compliance guide ensures that your SPARC data architecture not only meets current regulatory requirements but is also prepared for emerging privacy regulations worldwide. The modular approach allows for jurisdiction-specific customization while maintaining a consistent privacy-first foundation.
