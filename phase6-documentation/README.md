# Phase 6 — Documentation & Portfolio Publishing

**Status:** Pending (starts after Phase 5 completes)

---

## Objective

Transform all lab work into professional, publishable portfolio artifacts.
This phase is not less important than the technical phases — it is where
raw skill becomes visible evidence for hiring managers.

---

## Planned Deliverables

### GitHub Repository
- Complete README with architecture diagram, detection coverage table,
  key findings, and tech stack
- All phase folders populated with notes, commands, and screenshots
- Troubleshooting log fully documented (5 incidents already complete)
- ATT&CK Navigator coverage map exported and embedded

### Incident Reports (3 formal reports)
Written using SANS IR report template, covering:
- Executive Summary
- Timeline
- Technical Details
- Affected Systems
- MITRE ATT&CK Mapping
- Recommendations

Reports to cover the most interesting Phase 3 attack chains:
- T1110 Brute Force -> T1021.001 Lateral Movement
- T1003.001 LSASS Dump -> credential use
- T1486 Ransomware precursor chain

### Architecture Diagram
draw.io diagram showing full lab topology at app.diagrams.net.
Export as PNG and embed in README.

### Video Walkthrough
3-5 minute screen recording showing:
- Lab topology overview
- Attack simulation running from Kali
- Alert firing in Wazuh in real time
- Custom detection rule demonstrating Phase 4 work

### CV and LinkedIn Updates
- CV bullet points updated with results-oriented language
- LinkedIn post: #SOC #cybersecurity #blueteam #MITREATTACK #Lesotho

---

## Commit History Goal

A recruiter scrolling the GitHub commit history should see 14+ weeks of
dated, descriptive commits — proof of sustained, independent work.

Commit message format used throughout:
- Add incident 005 - wazuh cert mismatch fix
- Add T1046 attack report - partial detection documented
- Update README - phase 2 complete
- Add T1110 brute force report - detection confirmed
