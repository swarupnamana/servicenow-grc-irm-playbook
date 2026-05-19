
# 🛡️ ServiceNow GRC / IRM Implementation Playbook

> A comprehensive implementation guide and pattern library for ServiceNow Governance, Risk & Compliance (GRC) and Integrated Risk Management (IRM) — built from 7+ years of enterprise delivery experience.

<p>
  <img src="https://img.shields.io/badge/ServiceNow-GRC%2FIRM-green?style=for-the-badge&logo=servicenow&logoColor=white"/>
  <img src="https://img.shields.io/badge/Scope-Enterprise-1c2b3a?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Author-Swarup%20Kumar%20Namana-b8922a?style=for-the-badge"/>
</p>

---

## 📌 Overview

This playbook documents battle-tested implementation patterns, architecture decisions, and automation strategies for ServiceNow GRC/IRM deployments. It covers the full compliance lifecycle — from scoped application design to automated control testing, risk scoring, audit management, and vendor risk assessment.

**Key outcomes this playbook achieves:**
- ✅ Full audit readiness across enterprise environments
- ✅ Automated control testing reducing manual effort by 60%+
- ✅ Consistent risk scoring across business units
- ✅ Streamlined vendor risk questionnaire workflows
- ✅ Real-time compliance dashboards for executive visibility

---

## 📂 Repository Structure

```
servicenow-grc-irm-playbook/
│
├── 01-architecture/
│   ├── scoped-app-design.md          # Scoped application design principles
│   ├── domain-separation.md          # Domain separation for multi-entity orgs
│   └── csdm-alignment.md             # CSDM alignment for GRC entities
│
├── 02-policy-compliance/
│   ├── policy-lifecycle.md           # Policy authoring → approval → publish flow
│   ├── control-framework.md          # Control design and mapping patterns
│   ├── control-testing-automation.md # Automated control testing approach
│   └── exception-workflow.md         # Policy exception request workflow
│
├── 03-risk-management/
│   ├── risk-scoring-model.md         # Risk scoring methodology & formulas
│   ├── risk-assessment-workflow.md   # Risk assessment lifecycle
│   ├── issue-remediation.md          # Issue tracking & remediation workflow
│   └── risk-acceptance.md            # Risk acceptance & sign-off patterns
│
├── 04-audit-management/
│   ├── audit-lifecycle.md            # Audit planning → fieldwork → reporting
│   ├── audit-entity-mapping.md       # Audit entity & scope configuration
│   └── finding-remediation.md        # Audit finding & remediation tracking
│
├── 05-vendor-risk/
│   ├── vendor-onboarding.md          # Vendor onboarding & classification
│   ├── questionnaire-design.md       # VRM questionnaire design patterns
│   └── vendor-scoring.md             # Vendor risk scoring & tiering
│
├── 06-integrations/
│   ├── cmdb-grc-mapping.md           # CMDB → GRC entity relationship mapping
│   └── itsm-grc-bridge.md            # ITSM incident → GRC issue automation
│
├── 07-reporting/
│   ├── pa-dashboards.md              # Performance Analytics setup for GRC
│   ├── compliance-scorecard.md       # Executive compliance scorecard design
│   └── kpi-indicators.md             # KPI indicators & thresholds
│
└── 08-templates/
    ├── tdd-template.md               # Technical Design Document template
    ├── testing-checklist.md          # UAT checklist for GRC modules
    └── go-live-checklist.md          # Go-live readiness checklist
```

---

## 🏗️ Architecture Principles

### Scoped Application Design
Always deploy GRC customizations within a scoped application to ensure:
- **Upgrade safety** — customizations survive platform upgrades
- **Clear ownership** — all GRC artifacts grouped under one scope
- **Delegation control** — scoped app roles separate from global admin

```
Recommended Scoped App Structure:
├── Scope Name:     x_org_grc_custom
├── Tables:         Extend base GRC tables (never modify OOB)
├── Roles:          x_org_grc_custom.admin / .analyst / .viewer
└── Update Sets:    One per release cycle, named: GRC_v1.0_YYYYMMDD
```

### Entity Hierarchy
```
Organization
└── Business Unit
    └── Department
        └── Process
            ├── Policies      ──► Controls ──► Control Tests
            ├── Risks         ──► Issues   ──► Remediation Tasks
            └── Audit Entities ──► Findings ──► Action Plans
```

---

## ⚙️ Control Testing Automation

### Pattern: Automated Control Testing via Scheduled Jobs

One of the highest-value automations in GRC is replacing manual control tests with automated evidence collection.

**Implementation Approach:**

```javascript
// Script Include: GRCControlTestAutomator
var GRCControlTestAutomator = Class.create();
GRCControlTestAutomator.prototype = {
    initialize: function() {},

    // Automatically evaluate control effectiveness
    evaluateControl: function(controlTestSysId) {
        var ct = new GlideRecord('sn_grc_control_test');
        if (!ct.get(controlTestSysId)) return false;

        var result = this._runEvidenceCheck(ct);

        ct.test_result = result.passed ? 'effective' : 'ineffective';
        ct.evidence = result.evidence;
        ct.state = 'complete';
        ct.update();

        // Auto-create issue if control failed
        if (!result.passed) {
            this._createIssue(ct, result.findings);
        }

        return result;
    },

    // Evidence check logic (customize per control type)
    _runEvidenceCheck: function(ct) {
        var evidence = [];
        var passed = true;

        // Example: Check if change approval rate meets threshold
        var gr = new GlideAggregate('change_request');
        gr.addQuery('approval', 'approved');
        gr.addAggregate('COUNT');
        gr.query();

        if (gr.next()) {
            var approvedCount = gr.getAggregate('COUNT');
            evidence.push('Approved changes: ' + approvedCount);
            passed = (approvedCount > 0);
        }

        return { passed: passed, evidence: evidence.join('\n'), findings: [] };
    },

    // Auto-generate GRC Issue on test failure
    _createIssue: function(ct, findings) {
        var issue = new GlideRecord('sn_grc_issue');
        issue.initialize();
        issue.short_description = 'Control Test Failed: ' + ct.getDisplayValue('control');
        issue.description = findings.join('\n');
        issue.state = 'open';
        issue.risk_rating = 'high';
        issue.source = ct.sys_id;
        issue.insert();
    },

    type: 'GRCControlTestAutomator'
};
```

---

## 📊 Risk Scoring Model

### Quantitative Risk Scoring Formula

```
Inherent Risk Score  = Likelihood (1-5) × Impact (1-5)
Residual Risk Score  = Inherent Risk × (1 - Control Effectiveness %)
Risk Rating:
  1–5   → Low
  6–12  → Medium
  13–19 → High
  20–25 → Critical
```

### Risk Matrix

```
         │  Rare(1)  │Unlikely(2)│Possible(3)│ Likely(4) │AlmostCert(5)
─────────┼───────────┼───────────┼───────────┼───────────┼─────────────
Critical │     5     │    10     │    15     │    20     │     25
Major    │     4     │     8     │    12     │    16     │     20
Moderate │     3     │     6     │     9     │    12     │     15
Minor    │     2     │     4     │     6     │     8     │     10
Insignif │     1     │     2     │     3     │     4     │      5
```

---

## 📋 Policy Exception Workflow

```
Employee Request
      │
      ▼
Exception Form Submitted
(Justification + Duration + Approver)
      │
      ▼
Risk Owner Review ──► Rejected ──► Notify Requestor
      │
   Approved
      │
      ▼
Compliance Review ──► Rejected ──► Notify Requestor
      │
   Approved
      │
      ▼
Exception Granted (Time-bound)
      │
      ▼
Scheduled Review at Expiry ──► Renew or Close
```

---

## 🏢 Vendor Risk Management

### Vendor Tiering Model

| Tier | Risk Level | Review Frequency | Questionnaire |
|------|-----------|-----------------|---------------|
| Tier 1 | Critical | Quarterly | Full (120 questions) |
| Tier 2 | High | Semi-Annual | Standard (60 questions) |
| Tier 3 | Medium | Annual | Lite (25 questions) |
| Tier 4 | Low | Bi-Annual | Basic (10 questions) |

### Vendor Onboarding Automation

```
New Vendor Request
      │
      ▼
Auto-classify vendor tier (based on data access + spend)
      │
      ▼
Generate & Send appropriate questionnaire
      │
      ▼
Vendor completes questionnaire (portal)
      │
      ▼
Auto-score responses → Calculate Vendor Risk Score
      │
      ▼
Risk Score > Threshold?
  ├── Yes → Flag for manual review → Risk Owner Action
  └── No  → Auto-approve → Add to approved vendor registry
```

---

## 📈 Performance Analytics — GRC KPIs

| KPI | Formula | Target |
|-----|---------|--------|
| Control Effectiveness Rate | Effective Tests / Total Tests × 100 | > 90% |
| Policy Compliance Rate | Compliant Controls / Total Controls × 100 | > 95% |
| Risk Remediation SLA | Issues closed on time / Total Issues × 100 | > 85% |
| Vendor Risk Coverage | Vendors assessed / Total Vendors × 100 | 100% |
| Audit Finding Closure Rate | Closed Findings / Total Findings × 100 | > 80% |
| Open High Risks | Count of open risks rated High or Critical | < 5 |

---

## ✅ UAT Checklist

- [ ] Policy creation, approval, and publish workflow functions correctly
- [ ] Control tests trigger on schedule and auto-complete
- [ ] Failed control tests auto-generate issues with correct risk rating
- [ ] Risk scoring formula calculates inherent and residual correctly
- [ ] Vendor questionnaire sends automatically on tier classification
- [ ] Audit entity scoping and fieldwork assignment works end-to-end
- [ ] Exception workflow routes to correct approvers
- [ ] Performance Analytics dashboards populate with live data
- [ ] All email notifications fire with correct content
- [ ] ACLs prevent unauthorized access to GRC records

---

## 🚀 Go-Live Checklist

- [ ] Scoped app locked and update set packaged
- [ ] All scheduled jobs verified in production timing
- [ ] Admin and analyst roles assigned to correct groups
- [ ] Executive dashboard shared with leadership
- [ ] Training completed for GRC analysts
- [ ] Hypercare support plan in place (30 days post go-live)

---

## 👤 Author

**Swarup Kumar Namana**
Senior ServiceNow Developer & Platform Architect
Columbus, Ohio, USA

[![Portfolio](https://img.shields.io/badge/Portfolio-swarup--namana.netlify.app-b8922a?style=flat-square)](https://swarup-namana.netlify.app)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-swarupnamana-0077B5?style=flat-square&logo=linkedin)](https://linkedin.com/in/swarupnamana)
[![Email](https://img.shields.io/badge/Email-swarupnamana03%40gmail.com-D14836?style=flat-square&logo=gmail)](mailto:swarupnamana03@gmail.com)

---

*This playbook is based on real-world enterprise implementations. All client data has been abstracted. Patterns are generalized for reuse across organizations.*
