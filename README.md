
# StegaShield Detection Validation Pilot

### Independent Detection Validation | SOC Investigation | Reproducibility

## Overview

This project documents my independent hands-on evaluation of StegaShield, an image steganography detection tool, within a controlled cybersecurity laboratory.

My goal was to investigate how StegaShield responds to known-clean and genuinely modified images, determine whether its classifications match independently established ground truth, and examine how endpoint and network telemetry can support a SOC investigation.

I conducted the pilot over seven practical stages, now organized into eight GitHub documentation phases. The eighth phase consolidates the final technical assessment.

The project focuses on practical testing, investigation reasoning, reproducibility, and evidence-backed conclusions rather than simply demonstrating that a security tool works.

## Research Objectives

- Build and verify a reliable SOC laboratory.
- Create an independent dataset with known ground truth.
- Deploy and validate the StegaShield detection API.
- Test genuine RGB-LSB steganographic modifications.
- Investigate incorrect model classifications.
- Correlate endpoint, network, and application evidence.
- Verify reproducibility and evidence integrity.
- Document the findings, limitations, and recommendations.

## Laboratory Environment

| Component | Technology | Role |
|---|---|---|
| Host | MacBook Air M2 | Laboratory workstation |
| Endpoint | Windows 11 ARM64 | Controlled testing activity |
| Linux Server | Ubuntu ARM64 | Detection service and network monitoring |
| SIEM | Splunk Enterprise | Centralized investigation |
| Endpoint Telemetry | Sysmon | Process and network events |
| Network Monitoring | Zeek | Network connection metadata |
| Reverse Proxy | Nginx | HTTPS API access |
| Detection Engine | StegaShield | Image analysis |
| Container Runtime | Docker | StegaShield deployment |
| Dataset Generation | Python and Pillow | Independent image generation |

## Project Roadmap

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

Each phase contains its own methodology, investigation findings, and selected supporting evidence.

## Overall Findings

The pilot produced two primary detection outcomes:

| Sample | Ground Truth | Prediction | Outcome |
|---|---|---|---|
| Clean reference image | CLEAN | clean | True Negative |
| RGB-LSB modified image | MODIFIED | clean | False Negative |

StegaShield correctly classified the known-clean image but failed to identify the corresponding image containing a controlled 23-byte RGB-LSB payload.

The embedded payload was subsequently recovered, confirming that the modified image contained the expected data.

The SOC investigation also demonstrated how Sysmon, Splunk, Zeek, and Nginx records could be correlated to reconstruct authorized image-submission activity.

These findings apply to the tested samples and environment. They do not establish the detection engine's overall accuracy or performance across other datasets.

## Investigation Approach

The project follows an evidence-driven methodology:

**Baseline → Controlled Testing → Observation → Investigation → Correlation → Verification → Documentation**

Every reported finding must be supported by actual laboratory evidence.

Only selected screenshots that demonstrate a meaningful test, observation, investigation, or verification result will be included in the relevant phase.

## Project Limitations

This was a small-scale, controlled research pilot.

The primary validation used one synthetic clean image and one corresponding RGB-LSB-modified image.

Broader evaluation would require additional independent samples, natural images, different payload sizes, and alternative embedding methods.

The experiment does not establish production readiness, native StegaShield SIEM integration, or malicious activity.

## Project Conclusion

This investigation demonstrated the importance of establishing independent ground truth, verifying telemetry, investigating unexpected results, and preserving reproducible evidence.

The central finding was a confirmed false negative for a controlled RGB-LSB-modified image.

The project also reinforced the value of correlating multiple SOC telemetry sources to support defensible technical conclusions.

The detailed methodology, screenshots, observations, and discoveries are documented separately within the eight project phases.

---

**Independent Research | Detection Validation | SOC Investigation**
