
# StegaShield Detection Validation Pilot

**Independent Detection Validation | SOC Investigation | Reproducibility**

An evidence-driven cybersecurity research project evaluating StegaShield's image steganography detection behavior within a controlled SOC laboratory.

## Project Overview

I conducted this independent pilot to investigate how StegaShield classifies known-clean and genuinely steganographically modified images.

The goal was not simply to demonstrate a working detection tool. I wanted to establish independent ground truth, test the engine under controlled conditions, investigate unexpected results, and verify whether the associated activity could be reconstructed using SOC telemetry.

The original hands-on pilot was structured across seven days. For GitHub documentation, I organized the completed work into eight phases, with the final phase serving as the consolidated technical assessment.

Each phase documents its own investigation methodology, observations, discoveries, and supporting evidence where available.

## Laboratory Architecture

The pilot used a Mac M2 host running Splunk Enterprise, together with Windows 11 and Ubuntu ARM64 virtual machines.

The Windows endpoint generated controlled activity and provided Sysmon telemetry. Ubuntu hosted the image-analysis environment and network monitoring components.

![StegaShield Detection Validation Lab Architecture](assets/lab-architecture.png)

**Architecture note:** This is the original laboratory design diagram. It includes both operational components and initially planned integrations. Components shown in the diagram should not automatically be interpreted as successfully integrated or used during the final experiment.

The verified monitoring and testing paths are documented in the relevant phases.

### Core Technologies

| Technology | Purpose |
|---|---|
| Windows 11 ARM64 | Controlled endpoint activity |
| Ubuntu 24.04 ARM64 | Detection service and network analysis |
| Splunk Enterprise | Centralized SOC investigation |
| Sysmon | Windows process and network telemetry |
| Splunk Universal Forwarder | Windows log forwarding |
| Zeek | Network connection metadata |
| Docker | StegaShield container deployment |
| Nginx | HTTPS reverse proxy |
| Python and Pillow | Independent image generation and modification |

## Research Objectives

The project focused on the following questions:

1. Can StegaShield correctly classify a known-clean image?
2. Can it identify an image containing a genuine RGB-LSB modification?
3. How do the model's probabilities differ between matching clean and modified images?
4. Can an embedded payload be independently verified and recovered?
5. Can Sysmon, Splunk, Zeek, and Nginx evidence support investigation of controlled image submissions?
6. Are the test artifacts and findings reproducible?
7. What limitations should be considered before drawing broader conclusions?

## Project Phases

The complete investigation is organized into eight phases.

| Phase | Investigation | Documentation |
|---|---|---|
| 01 | Environment & Laboratory Baseline | [View Phase 01](phase-01-environment-baseline/README.md) |
| 02 | Independent Dataset & Ground Truth | [View Phase 02](phase-02-independent-dataset/README.md) |
| 03 | StegaShield Deployment & Clean Baseline | [View Phase 03](phase-03-deployment-baseline/README.md) |
| 04 | RGB-LSB Detection Validation | [View Phase 04](phase-04-lsb-detection-validation/README.md) |
| 05 | Error Analysis & Generalization Limits | [View Phase 05](phase-05-error-analysis/README.md) |
| 06 | HTTPS & SOC Telemetry Correlation | [View Phase 06](phase-06-soc-telemetry-correlation/README.md) |
| 07 | Reproducibility & Evidence Integrity | [View Phase 07](phase-07-reproducibility-integrity/README.md) |
| 08 | Consolidated Technical Assessment | [View Phase 08](phase-08-final-assessment/README.md) |

### Phase 01 — Environment & Laboratory Baseline

Established the laboratory infrastructure and verified endpoint and network monitoring.

Investigated missing Sysmon Event ID 3 telemetry and verified network event collection in Splunk following remediation.

**Documentation:** Text-only technical investigation.

### Phase 02 — Independent Dataset & Ground Truth

Created an independently generated image dataset using Python and Pillow.

Established clean and RGB-LSB-modified samples with known labels, controlled payloads, and SHA-256 fingerprints.

### Phase 03 — StegaShield Deployment & Clean Baseline

Deployed StegaShield on Ubuntu using Docker.

Verified the detection service and established a baseline classification using a known-clean image.

### Phase 04 — RGB-LSB Detection Validation

Evaluated StegaShield against an image containing a genuine 23-byte RGB-LSB embedded payload.

Recorded the model's response and compared it against independent ground truth.

### Phase 05 — Error Analysis & Generalization Limits

Investigated the observed false-negative classification.

Compared model probabilities, reviewed repeated analysis records, and documented the limitations of the controlled experiment.

### Phase 06 — HTTPS & SOC Telemetry Correlation

Performed authorized HTTPS image submissions from Windows to Ubuntu.

Investigated process, network, and application activity using Sysmon, Splunk, Zeek, and Nginx.

### Phase 07 — Reproducibility & Evidence Integrity

Reviewed image-generation and reproduction scripts, verified SHA-256 checksums, recovered the embedded payload, and confirmed temporary-file cleanup.

### Phase 08 — Consolidated Technical Assessment

Consolidates the investigation findings, evidence references, detection results, SOC observations, limitations, and recommendations into a final technical assessment.

**This is a documentation phase, not an additional experiment.**

---

## Key Research Findings

The controlled test produced the following detection outcomes:

| Sample | Ground Truth | Prediction | Result |
|---|---|---|---|
| Clean reference image | CLEAN | clean | True Negative |
| RGB-LSB modified image | MODIFIED | clean | False Negative |

StegaShield correctly classified the clean reference image but failed to identify the corresponding image containing a controlled 23-byte RGB-LSB payload.

The modified image was classified as `clean`, despite independently established ground truth and subsequent successful payload recovery.

The investigation also demonstrated how multiple SOC telemetry sources could be correlated to reconstruct the authorized image submissions.

**These findings apply to the specific samples and tested deployment. They do not establish overall StegaShield accuracy, production reliability, or performance across other steganography methods.**

## Investigation Methodology

The pilot followed an evidence-driven workflow:

**Baseline → Controlled Activity → Observation → Investigation → Correlation → Verification → Documentation**

Each phase distinguishes:

- What I intended to test
- What I actually performed
- What the tools returned
- What I discovered
- What the evidence supports
- What remains unverified

The individual phase READMEs contain the detailed methodology, technical observations, investigation findings, and relevant supporting evidence.

## Evidence & Reproducibility

The project uses genuine laboratory evidence, including:

- Independent ground-truth records
- Python generation and reproduction scripts
- Original and modified image artifacts
- SHA-256 integrity verification
- Saved StegaShield API responses
- Windows Sysmon events
- Splunk investigation results
- Zeek network records
- Nginx HTTPS access logs
- Docker deployment metadata
- Selected screenshots of completed laboratory activities

### Evidence Organization

**Phase 01 — Text-Only Documentation**

Phase 01 is intentionally documented without screenshots. Its README preserves the laboratory baseline, Sysmon troubleshooting, Splunk verification, technical findings, and limitations.

**Phases 02–07 — Technical Documentation and Selected Screenshots**

Each phase contains its own README and an `evidence/` directory with carefully selected, genuine laboratory screenshots.

Only screenshots directly supporting a relevant action, observation, investigation, or verification result are included.

The original screenshot archives are source material and are not automatically the final published evidence set.

**Phase 08 — Consolidated Technical Assessment**

Phase 08 brings together the completed research findings, evidence references, SOC observations, and recommendations.

It references previous phases rather than duplicating their screenshots.

### Evidence Integrity

The reproducibility review confirmed:

| Verification | Result |
|---|---|
| Original and reproduced image checksum checks | 4/4 PASS |
| Saved analysis JSON checksum checks | 3/3 PASS |
| Embedded 23-byte payload recovery | PASS |
| Docker image identity inspection | Verified |
| Temporary transfer directory cleanup | Verified |

These results establish agreement with the recorded evidence and checksums under the tested conditions.

They do not independently establish a complete forensic chain of custody.

Detailed verification is documented in [Phase 07](phase-07-reproducibility-integrity/README.md).

## Scope & Limitations

This was a controlled, non-production research pilot.

The primary validation used one synthetic clean image and one corresponding RGB-LSB-modified image.

The results do not establish performance across natural photographs, broader image datasets, additional embedding techniques, or different payload sizes.

Dataset A was identified as external reference material and was reportedly associated with the product team's training/testing workflow. It was not used to establish independent generalization.

The experiment did not establish native StegaShield-to-Splunk integration.

Encrypted network connection metadata was investigated separately from the application-level image analysis.

A steganography classification does not independently establish malicious activity or data exfiltration.

The model's internal architecture, training process, and learned features were not independently audited.

## Final Conclusion

The pilot demonstrated one correct clean-image classification and one missed detection of a controlled RGB-LSB modification.

The project also demonstrated the importance of verifying endpoint telemetry, correlating multiple evidence sources, and preserving reproducible testing artifacts.

The central lesson was that meaningful detection validation requires independent ground truth, controlled experimentation, and conclusions supported by actual evidence.

For the complete technical investigations, selected screenshots, discoveries, and research limitations, explore the individual phases above.

---

**Independent Cybersecurity Research | SOC Investigation | Detection Validation**

