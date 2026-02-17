# CMMC / NIST SP 800-171 Audit & Compliance System — Project Writeup

**Status:** Active Development

---

## Table of Contents

1. [How Was This Performed?](#1-how-was-this-performed)
2. [What Techniques Were Used, and Why?](#2-what-techniques-were-used-and-why)
3. [How Does a DIB Contractor Use This to Manage CMMC and NIST 800-171 Risks?](#4-how-does-a-dib-contractor-use-this-to-manage-cmmc-and-nist-800-171-risks-from-unfulfilled-obligations)
4. [How Can a DIB Contractor Determine the Best Course for Remediation?](#5-how-can-a-dib-contractor-use-this-to-know-what-the-best-course-for-remediation-action-is)
5. [How Can Other NIST SP 800 Publications Enhance Results?](#6-how-can-a-dib-contractor-use-other-nist-sp-800-works-to-enhance-their-results-from-using-this-application)
6. [What Was Unsuccessful?](#7-what-was-unsuccessful)
7. [Why Was This Successful?](#8-why-was-this-successful)

---

## 1. How Was This Performed?

### The Problem

Defense Industrial Base (DIB) contractors that handle Controlled Unclassified Information (CUI) are subject to DFARS 252.204-7012, which mandates implementation of the 110 security controls defined in NIST SP 800-171 Rev 2. With the phased rollout of the Cybersecurity Maturity Model Certification (CMMC) 2.0 program, these contractors now face formal third-party assessment by CMMC Third-Party Assessment Organizations (C3PAOs) for Level 2 certification.

Having delivered services to more than 40 active client engagements in the DIB, I observed a consistent pattern: contractors struggle not with understanding *what* they need to do, but with *tracking where they are, what's left, and what to prioritize*. The compliance lifecycle; assess, document, remediate, and report was being managed through fragmented spreadsheets, disconnected SharePoint folders, and inconsistent SSP documents. This fragmentation introduced risk: controls fell through the cracks, evidence went unmapped, and POA&M items stagnated without accountability.

This project was built to solve that problem.

### Development Approach

The system was developed iteratively, starting from the data layer and building upward:

**Phase 1: Data Foundation**
The first step was modeling the complete NIST SP 800-171 Rev 2 control catalog as structured data. I created a JSON representation of all 110 controls across 14 families, including metadata fields for CMMC level alignment, DFARS applicability, and cross-references to the parent NIST SP 800-53 controls. This catalog serves as the single source of truth that the entire application operates against.

**Phase 2: Domain Models**
I defined Python data classes for every entity in the compliance lifecycle: Controls, Assessments, Evidence Artifacts, POA&M Items, and Organization Profiles. Each model includes serialization and deserialization methods to support JSON persistence to ensure the entire state of an assessment can be saved, loaded, transferred between sessions, or handed off to another assessor.

**Phase 3: Scoring Engine**
The SPRS (Supplier Performance Risk System) scoring engine was implemented using the actual DoD NIST SP 800-171 Assessment Methodology control weights. Each of the 110 controls carries a weight of 1, 3, or 5 based on its criticality to CUI protection. The engine calculates deductions for unimplemented or partially implemented controls against a maximum score of 110, producing a score range of -203 to 110. This matches the methodology that DoD uses for DFARS 252.204-7019/7020 self-assessments and that DIBCAC uses for medium/high assessments.

**Phase 4: Application Layer**
The Streamlit web framework was selected to provide an interactive dashboard without the overhead of a full frontend/backend separation. The application was structured as a multi-page Streamlit app with four functional pages (Control Assessment, Evidence Manager, POA&M Tracker, Export & Reports) plus a main dashboard for scoring and state management.

**Phase 5: Testing & Validation**
A test suite was written for the scoring engine, which is the most critical component from an accuracy standpoint. Eight tests validate perfect scores, maximum deductions, single-control deductions, Not Applicable handling, and partial implementation scoring behavior. All tests pass.

### Technology Selection Rationale

| Decision | Choice | Rationale |
|---|---|---|
| Language | Python 3.10+ | Dominant in security tooling, data processing, and the compliance automation space. Familiar to the target user base. |
| Framework | Streamlit | Rapid prototyping, built-in state management, zero JavaScript required. Allows a compliance officer — not a web developer — to understand and extend the tool. |
| Data Storage | JSON files | No database dependencies. Portable. A compliance officer can email an assessment file. Aligns with how DIB contractors actually transfer compliance artifacts. |
| Control Catalog | Structured JSON | Machine-readable, versionable, diffable. Enables automated scoring and cross-referencing without parsing PDFs. |
| Scoring | Custom engine with DoD weights | No existing open-source implementation accurately applies the DoD Assessment Methodology's per-control weighting. This had to be built from primary sources. |

---

## 2. What Techniques Were Used, and Why?

### Structured Control Decomposition

The NIST SP 800-171 Rev 2 control set is organized into 14 families (Access Control, Awareness & Training, Audit & Accountability, etc.), but the relationships between controls, their parent 800-53 controls, and their CMMC practice mappings are not natively represented in a machine-readable format. I decomposed the entire 110-control set into a structured JSON schema that preserves these relationships, enabling programmatic filtering, searching, cross-referencing, and scoring.

This technique matters because compliance is inherently a data problem. When a contractor asks, "Which controls in the System and Communications Protection family are still unassessed?", the answer should be instantaneous and not require a manual scan through a 300-page SSP.

### Weighted Risk Scoring (SPRS Methodology)

The scoring engine implements the DoD's own weighting system rather than treating all 110 controls as equal. This is critical because the DoD does not treat them equally. A failure to implement multi-factor authentication (3.5.3, weight 5) has a fundamentally different risk profile than a failure to obscure authentication feedback (3.5.11, weight 1). The SPRS score directly impacts a contractor's ability to win contracts and is reported in the Supplier Performance Risk System and is visible to contracting officers.

By encoding these weights, the tool enables contractors to immediately see the *score impact* of each gap, which directly informs prioritization decisions.

### Inherited Control Tracking

In practice, many DIB contractors rely on Managed Service Providers (MSPs), Managed Security Service Providers (MSSPs), or Cloud Service Providers (CSPs) to implement a subset of their controls. These are "inherited controls"; meaning that the contractor doesn't implement them directly but relies on a service provider's shared responsibility model.

The assessment interface includes explicit fields for marking controls as inherited and identifying the source. This matters because C3PAOs and DIBCAC assessors will specifically ask, "How did you validate that your MSP actually implements this control?" A claim of inheritance without validation is a common audit failure point. The tool forces the assessor to be explicit about what's inherited and from whom, creating a documentation trail that survives scrutiny.

### Evidence-to-Control Mapping

Rather than maintaining a flat list of evidence artifacts, the system implements a many-to-many mapping between evidence and controls. A single Access Control Policy might satisfy aspects of 3.1.1, 3.1.2, 3.1.5, and 3.1.22. Conversely, a single control might require multiple pieces of evidence (a policy, a screenshot of the configuration, and a log sample showing enforcement).

This mapping technique enables two critical capabilities: forward coverage analysis (which controls lack evidence?) and reverse traceability (which artifacts support this specific control?). Both are essential for audit readiness.

### Auto-Generated POA&M from Assessment Gaps

Manual POA&M creation is tedious and error-prone. The system auto-generates POA&M items by scanning the assessment data for any control marked as Not Implemented or Partially Implemented, then creating a structured finding with the gap description, risk level, and control reference pre-populated. The assessor then fills in milestones, responsible parties, and dates.

This technique eliminates the common problem of gaps identified during assessment that never make it into the POA&M which is itself a gap in gap-tracking analyses and processes.

### Stateless Persistence via JSON Serialization

The entire application state; organization profile, all 110 control assessments, all evidence artifacts, and all POA&M items are serialized to a single JSON file. This is a deliberate architectural decision. DIB contractors need to be able to save their work, close the application, reopen it next week, and pick up where they left off. They need to be able to email the assessment file to their CMMC consultant. They need to be able to keep it in a SharePoint folder alongside their SSP.

JSON files meet all of these requirements with zero infrastructure overhead.

---

## 3. How Does a DIB Contractor Use This to Manage CMMC and NIST 800-171 Risks from Unfulfilled Obligations?

### Understanding the Regulatory Landscape

A DIB contractor with an active DoD contract that involves CUI is subject to:

- **DFARS 252.204-7012** - Requires implementation of NIST SP 800-171
- **DFARS 252.204-7019** - Requires a self-assessment and SPRS score submission
- **DFARS 252.204-7020** - Requires cooperation with DoD assessments (DIBCAC)
- **CMMC 2.0 Level 2** - Requires third-party assessment by a C3PAO (for contracts involving CUI)

Failure to comply can result in loss of contract eligibility, False Claims Act liability, and reputational damage. The risk is not theoretical; cases have been pursued and judgements issued against contractors that misrepresented their compliance posture.

### Using the Tool to Manage These Risks

**Step 1: Establish Baseline (Dashboard + Control Assessment)**

The contractor opens the application and begins assessing each of the 110 controls. For each control, they select the implementation status:

- **Implemented** - The control is fully in place and operating as intended
- **Partially Implemented** - Some aspects are in place, but gaps remain
- **Not Implemented** - The control is not in place
- **Planned** - The control is not in place but a plan exists
- **Not Applicable** - The control does not apply to this system

The dashboard immediately reflects the SPRS score. If the contractor is at a score of 47, they now have a quantified understanding of their compliance posture and know exactly which controls are dragging the score down.

**Step 2: Quantify Gap Risk (Scoring Engine)**

The SPRS score is a weighted representation of risk. A contractor with a score of 85 who is missing 3.5.3 (MFA, weight 5) has a fundamentally different risk profile than one at 85 who is missing five weight-1 controls. The scoring engine's per-control weight visibility helps contractors understand which gaps carry the most regulatory and operational risk.

**Step 3: Document What Exists (Evidence Manager)**

For every control marked as Implemented or Partially Implemented, the contractor should have evidence. The Evidence Manager allows them to create artifact records mapped to specific controls, building an evidence repository that a C3PAO assessor can review. The coverage matrix shows at a glance which controls lack supporting evidence and will allow gap remediation to occur before it's flagged during a formal assessment.

**Step 4: Plan Remediation (POA&M Tracker)**

For every control that is Not Implemented or Partially Implemented, the contractor needs to reflect it in the Plan of Action & Milestones. The auto-generation feature creates POA&M items from assessment gaps, pre-populated with the gap description and risk level. The contractor then adds milestones, responsible parties, target dates, and resource/cost estimates.

This is where unfulfilled obligations become managed risks. A POA&M is not an admission of failure, rather; it is a *documented plan* to achieve compliance. C3PAOs expect to see POA&Ms. What they do not accept is gaps without an acknowledged plan.

**Step 5: Report and Prepare (Export & Reports)**

The contractor exports their SSP summary, assessment CSV, POA&M CSV, and evidence inventory. These deliverables feed directly into the documentation package that a C3PAO reviews during a CMMC Level 2 assessment. The SSP summary includes the SPRS score, family-level implementation percentages, and per-control status thereby providing a comprehensive snapshot of the compliance posture.

### Ongoing Risk Management

Compliance is not a point-in-time event. The contractor should reassess monthly updating control statuses as remediation progresses; adding new evidence as controls are implemented, and closing POA&M items as milestones are achieved. The JSON save/load feature supports this iterative workflow by preserving the complete assessment state between sessions.

---

## 4. How Can a DIB Contractor Use This to Know What the Best Course for Remediation Action Is?

### Prioritization Framework

Not all gaps are created equal. The tool provides three dimensions for prioritization:

**Dimension 1: SPRS Weight (Score Impact)**

The scoring engine exposes the per-control weight (1, 3, or 5). A contractor at SPRS score 85 who needs to reach 100 should first remediate unimplemented weight-5 controls, then weight-3, then weight-1. This produces the maximum score improvement per remediation action.

Example prioritization:

| Control | Weight | Status | Score Impact if Remediated |
|---|---|---|---|
| 3.5.3 (MFA) | 5 | Not Implemented | +5 |
| 3.13.11 (FIPS crypto) | 5 | Not Implemented | +5 |
| 3.1.14 (Managed access points) | 3 | Not Implemented | +3 |
| 3.3.3 (Review logged events) | 1 | Not Implemented | +1 |

Remediating 3.5.3 and 3.13.11 first yields +10 points from two actions. Remediating ten weight-1 controls yields the same +10 but requires five times the effort.

**Dimension 2: Risk Level (Operational Impact)**

The assessment allows the assessor to assign a risk level (Critical, High, Moderate, Low) to each gap. This captures context that the SPRS weight alone does not. A weight-1 control might carry Critical risk in a specific environment. For example, 3.3.9 (limit audit log management to privileged users) is weight-3, but in an environment where audit logs are the primary detection mechanism, its operational risk may be Critical.

The POA&M tracker surfaces risk levels alongside status, allowing the contractor to sort and filter by risk to ensure Critical items are addressed first regardless of score weight.

**Dimension 3: Remediation Complexity and Cost**

The POA&M tracker includes fields for resources required and cost estimates. A contractor should consider the effort-to-impact ratio. Some high-weight controls can be remediated quickly and cheaply (enabling MFA is a configuration change). Others require significant investment (implementing a SIEM for 3.14.6 and 3.14.7 involves licensing, deployment, and ongoing operations).

### Recommended Prioritization Logic

```
Priority = (SPRS Weight × Risk Multiplier) / Estimated Remediation Effort

Where:
  Risk Multiplier: Critical = 4, High = 3, Moderate = 2, Low = 1
  Remediation Effort: Scale of 1 (config change) to 5 (major project)
```

A weight-5, Critical-risk control that requires only a configuration change (effort = 1) would score: (5 × 4) / 1 = 20. A weight-1, Low-risk control requiring a major project (effort = 5) would score: (1 × 1) / 5 = 0.2. The contractor remediates in descending priority order.

### Clustering Remediation by Family

The family-level progress bars on the dashboard reveal another pattern: if a contractor is at 0% implementation in Incident Response (3.6), that likely means they lack an incident response plan entirely. Rather than remediating controls individually the capability gap should be addressed by; developing an IR plan, training personnel, conducting tabletop exercises, all of which will simultaneously satisfy 3.6.1, 3.6.2, and 3.6.3.

The tool's family-level view enables this "capability cluster" approach to remediation, which is more efficient than a control-by-control methodology.

### Inherited Controls as a Remediation Strategy

For controls that are expensive or complex to implement in-house, the contractor should evaluate whether a qualified MSP, MSSP, or CSP can satisfy the requirement. The tool's inherited control tracking supports this strategy by documenting the inheritance claim and the source. However, the contractor must validate the claim; requesting the provider's SOC 2 Type II, reviewing their shared responsibility matrix, and confirming the specific control is covered. An unvalidated inheritance claim is worse than no claim at all.

---

## 5. How Can a DIB Contractor Use Other NIST SP 800 Works to Enhance Their Results from Using This Application?

NIST SP 800-171 does not exist in isolation. It is derived from NIST SP 800-53 and is supported by an ecosystem of companion publications. A contractor that leverages these resources will produce a more robust compliance posture and a more defensible assessment.

### NIST SP 800-53 Rev 5: Security and Privacy Controls

Every NIST SP 800-171 control maps to one or more NIST SP 800-53 controls (this mapping is included in the tool's control catalog). SP 800-53 provides significantly more detailed implementation guidance than 800-171. When a contractor is unsure *how* to implement a 800-171 control, they should consult the corresponding 800-53 control's supplemental guidance.

**Application:** The control catalog in this tool includes the `nist_800_53` cross-reference for each control. When assessing 3.5.3 (MFA), the contractor can reference IA-2(1) and IA-2(2) in SP 800-53 for detailed implementation expectations, including what constitutes an acceptable authentication factor.

### NIST SP 800-171A: Assessing Security Requirements

SP 800-171A defines the **assessment objectives** and **assessment methods** (examine, interview, test) for each of the 110 controls. This publication tells the contractor exactly what a C3PAO assessor will be looking for.

**Application:** Before marking a control as "Implemented" in this tool, the contractor should verify they can satisfy the assessment objectives defined in SP 800-171A. If SP 800-171A says the assessor will *test* that session locks activate after 15 minutes, the contractor needs evidence that the timeout is configured and operational and not just a policy that says it should be.

### NIST SP 800-37 Rev 2: Risk Management Framework (RMF)

SP 800-37 defines the overarching Risk Management Framework lifecycle: Prepare, Categorize, Select, Implement, Assess, Authorize, Monitor. While DIB contractors are not required to follow the full RMF process, the framework provides a structured methodology for managing security controls over time.

**Application:** The workflow in this tool (Assess → Document → Remediate → Report) aligns with the Assess and Monitor steps of the RMF. Contractors can use the RMF's "continuous monitoring" guidance (Step 7) to establish a cadence for reassessing controls using this tool to turn their compliance from a point-in-time event into an ongoing process.

### NIST SP 800-30 Rev 1: Guide for Conducting Risk Assessments

SP 800-30 provides a methodology for assessing risk to organizational operations, assets, and individuals. Control 3.11.1 specifically requires periodic risk assessments.

**Application:** The risk levels assigned to gaps in this tool (Critical, High, Moderate, Low) should be informed by a formal risk assessment process. SP 800-30 provides the methodology for determining risk based on threat likelihood and impact. A contractor that can articulate *why* a gap is rated Critical by citing specific threat sources and impact scenarios per SP 800-30 will produce a far more defensible POA&M than one that assigns risk levels subjectively.

### NIST SP 800-128: Guide for Security-Focused Configuration Management

Controls 3.4.1 through 3.4.9 (Configuration Management family) require baseline configurations, change tracking, and security impact analysis. SP 800-128 provides detailed guidance on implementing these controls.

**Application:** A contractor struggling with the CM family (common, given the operational overhead) can use SP 800-128 to develop a configuration management program that systematically satisfies all nine CM controls. This is a prime example of the "capability cluster" remediation approach by standing up the capability which will allow the controls to follow.

### NIST SP 800-61 Rev 2: Computer Security Incident Handling Guide

Controls 3.6.1 through 3.6.3 (Incident Response family) require an operational incident handling capability, tracking/reporting, and testing. SP 800-61 is the definitive guide for building that capability.

**Application:** A contractor that follows SP 800-61 to build their incident response capability will naturally satisfy the IR family controls. The guide covers preparation, detection and analysis, containment, eradication, recovery, and post-incident activity which maps directly to 3.6.1's requirement for "preparation, detection, analysis, containment, recovery, and user response activities.", and to specific DFARS 7012 requirements.  

### NIST SP 800-88 Rev 1: Guidelines for Media Sanitization

Control 3.8.3 requires sanitizing or destroying system media containing CUI before disposal or reuse. SP 800-88 defines the specific sanitization techniques (Clear, Purge, Destroy) appropriate for different media types.

**Application:** When a contractor assesses 3.8.3 and documents their implementation details in this tool, they should reference the specific SP 800-88 sanitization level they apply. A C3PAO assessor will expect the contractor to know the difference between "clearing" a drive (overwriting) and "purging" it (cryptographic erase or degaussing) and when each is appropriate.

### Additional Relevant Publications

| Publication | Relevance |
|---|---|
| **SP 800-63B** — Digital Identity Guidelines (Authentication) | Informs MFA implementation for 3.5.3, 3.5.4 |
| **SP 800-92** — Guide to Computer Security Log Management | Supports AU family controls (3.3.x) |
| **SP 800-115** — Technical Guide to Information Security Testing | Supports security assessment and vulnerability scanning (3.11.2, 3.12.1) |
| **SP 800-137** — Information Security Continuous Monitoring | Supports ongoing monitoring (3.12.3) |
| **SP 800-160 Vol 1** — Systems Security Engineering | Supports 3.13.2 (security architecture) |
| **SP 800-172** — Enhanced Security Requirements for CUI | Relevant for contractors moving toward CMMC Level 3 |

---

## 6. What Was Unsuccessful?

### PDF/DOCX Report Generation

The initial design included automated SSP and POA&M generation in PDF and Word formats that would match the templates commonly used in DIBCAC and C3PAO assessments. This was deferred because the formatting requirements for these documents vary significantly across C3PAOs and contracting organizations. A generic template would require customization for every engagement, negating the automation benefit. The current text-based SSP summary and CSV exports provide the data in formats that can be imported into any template.

**Lesson:** Don't automate formatting when the format isn't standardized. Automate the *data collection* and let the user apply their own formatting.

### OSCAL Integration

The Open Security Controls Assessment Language (OSCAL), developed by NIST, provides a standardized machine-readable format for security control catalogs, system security plans, assessment results, and POA&M documents. Implementing OSCAL import/export was on the initial roadmap but proved to be a significant scope expansion. The OSCAL schema is complex, and the tooling ecosystem around it is still maturing. Integration was deferred to a future release.

**Lesson:** Standards adoption is a project in itself. For an MVP, the priority should be functional completeness, not format compliance. OSCAL integration remains on the roadmap.

### Multi-Tenant Client Management

The original vision included the ability to manage multiple client assessments within a single instance thereby reflecting the my consultative workflow of managing simultaneous client engagements. Streamlit's session state model does not naturally support this. Implementing multi-tenancy would require a database backend, authentication, and role-based access control thereby fundamentally changing the architecture from a lightweight local tool to a hosted application.

**Lesson:** Know your MVP boundary. The single-assessment model serves individual contractors and consultants working with one client at a time. Multi-tenancy is a different product.

### Automated Vulnerability Scanner Integration

I explored integrating Nessus and Qualys scan results to automatically validate technical controls (3.11.2 — vulnerability scanning, 3.14.1 — flaw remediation, etc.). The challenge is that scan results require interpretation; Noting that a Nessus finding doesn't map 1:1 to a NIST 800-171 control and that the presence of a scan doesn't mean the control is "Implemented." Automated compliance determination from scan data introduces false confidence and false positives.

**Lesson:** Automation is powerful when the mapping is deterministic. Compliance assessment requires human judgment. The tool should support the human, not replace them.

---

## 7. Why Was This Successful?

### It Solves a Real Problem I Experienced Firsthand

This tool was not built from theory. It was built from the specific pain points observed while delivering compliance services to 40+ DIB contractors. Every feature addresses a real workflow failure I witnessed in practice: fragmented tracking, unmapped evidence, stagnant POA&Ms, inaccurate SPRS scores, and audit surprises from gaps that were identified but never formally documented.

The most successful tools are the ones their builders would actually use. I would use this.

### The Scoring Engine Is Accurate

The SPRS score matters. It is a number reported to the Department of War that directly impacts contract eligibility. The scoring engine in this tool uses the actual DoW Assessment Methodology control weights, validated by an eight-test suite covering edge cases (perfect score, zero assessments, N/A handling, partial implementation, single-control deduction). A compliance tool with an inaccurate scoring engine would put the contractor in a worse position.

### It Requires Zero Infrastructure

DIB contractors I've worked with are missing internal DevOps functions, personnel, or a team. They don't want to, or can't; deploy a database, configure a web server, or manage cloud infrastructure for a compliance tool. This application runs with `pip install` and `streamlit run`. The data saves to a JSON file they can put in a folder. This zero-infrastructure requirement is not a limitation; it is a deliberate design choice that aligns with the technical capabilities of the target user base.

### It Follows the Assessor's Mental Model

The application's structure mirrors how a C3PAO assessor actually conducts an assessment: review each control family, evaluate implementation status, verify evidence, document findings, generate a POA&M for gaps, and produce a report. A contractor that prepares using this tool is simultaneously building the documentation package the assessor will review. There is no translation step between "what the tool produces" and "what the assessor expects."

### It Enables Prioritization, Not Just Tracking

Most compliance spreadsheets track status. This tool enables *prioritization* through the SPRS weight system. When a contractor can see that remediating two weight-5 controls will improve their score by 10 points, while remediating ten weight-1 controls achieves the same result, they can allocate resources intelligently. This transforms compliance from a checklist exercise into active risk management and the intent of NIST SP 800-171.

### The Evidence Mapping Creates Audit Readiness

The many-to-many mapping between evidence and controls is the most underappreciated feature. In my experience, the single most time-consuming pre-audit activity was not assessing controls and *tieing the evidence* that supported the assessment together. By building evidence mapping into the assessment workflow rather than treating it as an afterthought, the tool ensures that when the C3PAO asks "Show me the evidence for 3.5.3," the answer is immediate.

### It Is Extensible

The modular architecture (separate modules for models, scoring, persistence, export, and catalog management) means the tool can grow. NIST SP 800-171 Rev 3 is on the horizon. CMMC practice-level overlays can be added as a mapping layer. OSCAL export can be implemented without touching the scoring engine. PDF generation can be added without modifying the assessment logic. The investment in clean separation of concerns pays dividends in maintainability and extensibility.

---

## References

- [NIST SP 800-171 Rev 2](https://csrc.nist.gov/publications/detail/sp/800-171/rev-2/final) — Protecting CUI in Nonfederal Systems
- [NIST SP 800-171A](https://csrc.nist.gov/publications/detail/sp/800-171a/final) — Assessing Security Requirements for CUI
- [NIST SP 800-53 Rev 5](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final) — Security and Privacy Controls
- [NIST SP 800-30 Rev 1](https://csrc.nist.gov/publications/detail/sp/800-30/rev-1/final) — Guide for Conducting Risk Assessments
- [NIST SP 800-37 Rev 2](https://csrc.nist.gov/publications/detail/sp/800-37/rev-2/final) — Risk Management Framework
- [DFARS 252.204-7012](https://www.acquisition.gov/dfars/252.204-7012) — Safeguarding Covered Defense Information
- [CMMC 2.0 Program](https://www.acq.osd.mil/cmmc/) — Cybersecurity Maturity Model Certification
- [DoD Assessment Methodology](https://www.acq.osd.mil/asda/dpc/cp/cyber/docs/safeguarding/NIST-SP-800-171-Assessment-Methodology-Version-1.2.1-6.24.2020.pdf) — NIST SP 800-171 DoD Assessment Methodology

---

*Part of my [project portfolio](https://github.com/AbacusNin/project-portfolio).*
