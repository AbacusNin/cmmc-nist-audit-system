# 🛡️ CMMC / NIST SP 800-171 Audit & Compliance System

[![Python](https://img.shields.io/badge/Python-3.10+-blue?style=flat-square&logo=python)](https://python.org)
[![Streamlit](https://img.shields.io/badge/Streamlit-1.30+-red?style=flat-square&logo=streamlit)](https://streamlit.io)
[![NIST 800-171](https://img.shields.io/badge/NIST%20SP%20800--171-Rev%202-blue?style=flat-square)](https://csrc.nist.gov/publications/detail/sp/800-171/rev-2/final)
[![CMMC](https://img.shields.io/badge/CMMC-Level%202-green?style=flat-square)](https://www.acq.osd.mil/cmmc/)
[![Tests](https://img.shields.io/badge/Tests-8%20passed-brightgreen?style=flat-square)]()

> A web-based compliance assessment, evidence management, and audit preparation
> tool for Defense Industrial Base (DIB) contractors subject to NIST SP 800-171,
> CMMC Level 2, and DFARS 252.204-7012 requirements.

---

## 🎯 Purpose

DIB contractors pursuing or maintaining CMMC Level 2 certification need to assess
all **110 NIST SP 800-171 Rev 2 controls**, maintain evidence repositories, track
remediation plans, and generate audit-ready deliverables. This tool provides a
unified workflow for the entire compliance lifecycle:

1. **Assess** — Evaluate each control's implementation status
2. **Document** — Map evidence artifacts to controls
3. **Remediate** — Track gaps via POA&M items with auto-generation
4. **Score** — Calculate SPRS scores using DoD Assessment Methodology weights
5. **Export** — Generate SSP summaries, POA&M reports, and CSV deliverables

Built from direct experience and understanding client pain points from DIB consultation engagements.

> 📝 **[Read the full project writeup →](PROJECT_WRITEUP.md)** — methodology, techniques, and lessons learned

---

## ✨ Features

### Dashboard
- **SPRS Score** calculated from DoD Assessment Methodology control weights
- Implementation status breakdown across all 14 control families
- Risk distribution summary (Critical / High / Moderate / Low)
- Visual progress bars per family with percentage completion
- One-click save/load of complete assessment state

### Control Assessment
- All **110 NIST SP 800-171 Rev 2 controls** with full requirement text
- Per-control assessment: status, implementation details, gaps, risk level
- **Inherited control tracking** — flag controls inherited from MSPs/MSSPs/CSPs
- Filter by family, status, or free-text search
- Bulk assessment actions for rapid initial evaluation
- Cross-reference to NIST SP 800-53 controls

### Evidence Manager
- Create and catalog evidence artifacts (policies, screenshots, configs, logs, etc.)
- **Map artifacts to multiple controls** — one policy can satisfy many requirements
- Evidence coverage matrix showing which controls lack artifacts
- 15 artifact type categories aligned to common audit expectations
- Filter and search across the evidence inventory

### POA&M Tracker
- **Auto-generate POA&M items** from assessment gaps (one click)
- Manual POA&M creation with milestones, dates, responsible parties, and cost estimates
- Status workflow: Open → In Progress → Delayed → Completed → Closed
- Risk-level tracking per finding
- Filter by status for audit review

### Export & Reports
- **SSP Summary Report** — text-based System Security Plan with SPRS score
- **Assessment CSV** — all 110 controls with full assessment data
- **POA&M CSV** — formatted for DIBCAC/C3PAO review
- **Evidence Inventory CSV** — complete artifact catalog with control mappings
- Organization profile management (CAGE code, UEI, system boundary)

---

## 🏗️ Architecture

```
cmmc-nist-audit-system/
├── app.py                          # Main dashboard (Streamlit entry point)
├── pages/
│   ├── 1_📋_Control_Assessment.py  # Per-control assessment interface
│   ├── 2_📁_Evidence_Manager.py    # Evidence artifact tracking
│   ├── 3_📌_POAM_Tracker.py        # POA&M management
│   └── 4_📤_Export_Reports.py      # Report generation & exports
├── src/
│   ├── __init__.py
│   ├── models.py                   # Data models (Assessment, Evidence, POA&M, etc.)
│   ├── control_catalog.py          # NIST 800-171 control loader & search
│   ├── scoring.py                  # SPRS scoring engine with DoD weights
│   ├── persistence.py              # JSON save/load for assessment state
│   └── export.py                   # CSV and SSP report generation
├── data/
│   └── nist_800_171_controls.json  # Complete 110-control catalog
├── tests/
│   └── test_scoring.py             # Scoring engine test suite (8 tests)
├── exports/                        # Generated reports (gitignored)
├── config/
├── .streamlit/
│   └── config.toml                 # Streamlit theme configuration
├── ****
├── ****
├── ****
└── README.md
```

## 📊 SPRS Scoring Methodology

This tool implements the **DoD NIST SP 800-171 Assessment Methodology** for
calculating Supplier Performance Risk System (SPRS) scores:

- **Perfect score:** 110 (all controls implemented)
- **Minimum possible:** -203 (all controls unimplemented)
- Each control carries a **weight of 1, 3, or 5** based on criticality
- Partially implemented controls are scored the same as not implemented (per DoD methodology)
- Not Applicable controls do not incur deductions

Scores are used by DoD for supplier risk assessment and are required for
DFARS 252.204-7019/7020 compliance.

---

## 🔄 Typical Workflow

```
1. Set Organization Profile (Export page)
           │
2. Assess Controls (Control Assessment page)
    ├── Mark implementation status
    ├── Document implementation details
    ├── Identify gaps
    └── Flag inherited controls
           │
3. Add Evidence (Evidence Manager page)
    ├── Upload/reference artifacts
    ├── Map to controls
    └── Review coverage gaps
           │
4. Generate POA&M (POA&M Tracker page)
    ├── Auto-generate from gaps
    ├── Assign milestones & owners
    └── Track remediation progress
           │
5. Export Deliverables (Export page)
    ├── SSP Summary Report
    ├── Assessment CSV
    ├── POA&M CSV
    └── Evidence Inventory CSV
           │
6. Save State (Dashboard)
    └── Download assessment JSON for future sessions
```

---

## 📚 Frameworks & Standards Referenced

| Standard | Coverage |
|---|---|
| **NIST SP 800-171 Rev 2** | All 110 controls, 14 families |
| **CMMC Level 2** | Aligned to 800-171 control set |
| **DFARS 252.204-7012** | CUI protection requirements |
| **DFARS 252.204-7019/7020** | SPRS scoring & assessment |
| **NIST SP 800-53** | Cross-referenced per control |

---

## 🛣️ Roadmap

- [ ] NIST SP 800-171 Rev 3 control catalog
- [ ] CMMC Level 2 Practice-to-control mapping overlay
- [ ] OSCAL (Open Security Controls Assessment Language) import/export
- [ ] FedRAMP inheritance mapping templates
- [ ] Multi-tenant support for managing multiple client assessments
- [ ] PDF report generation (SSP, POA&M)
- [ ] Automated evidence gap analysis recommendations
- [ ] Integration with vulnerability scanning tools (Nessus, Qualys)

---

## 🤝 Contributing

Please open an issue to discuss proposed changes before submitting a pull request.

---

## 📄 License

This project is not licensed for open source and all rights are retained.

---

## ⚠️ Disclaimer

This tool is provided for **educational and professional use**. It does not
constitute legal or compliance advice. Organizations should work with qualified
assessors (C3PAOs, RPOs, or compliance professionals) for official CMMC
assessments. Control weights and scoring methodology are based on publicly
available DoD guidance and may be updated by DoD.

---

*Part of my [project portfolio](https://github.com/AbacusNin/project-portfolio).*
