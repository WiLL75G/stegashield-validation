
# Phase 08 — Consolidated Technical Assessment

**StegaShield Detection Validation Pilot**

**Focus:** Final detection assessment, SOC investigation findings, evidence verification, operational limitations, and recommendations.

[Back to Main Project](../README.md)

---

## 1. Executive Summary

I conducted an independent StegaShield Detection Validation Pilot to evaluate the detection engine's behavior against controlled clean and steganographically modified images.

The project also investigated whether the associated image-submission activity could be reconstructed using endpoint, network, and application telemetry.

Rather than relying on the detector's classifications as ground truth, I generated my own clean reference image and created a corresponding RGB-LSB-modified image containing a controlled 23-byte payload.

I then deployed StegaShield, examined its predictions, investigated the observed false negative, correlated the associated HTTPS traffic, and verified the integrity of the retained research artifacts.

### Principal Result

StegaShield correctly classified the known-clean image but classified the corresponding RGB-LSB-modified image as clean.

This produced:

- One true negative.
- One false negative.

The embedded 23-byte payload was subsequently recovered directly from the preserved modified PNG.

These findings demonstrate a missed detection for the specific controlled sample.

They do not establish StegaShield's overall detection accuracy, reliability, or performance against other embedding methods.

## 2. Research Objectives

The pilot addressed seven research objectives.

| ID | Objective | Outcome |
|---|---|---|
| OBJ-01 | Establish reliable laboratory telemetry | Verified |
| OBJ-02 | Create independent ground-truth samples | Verified |
| OBJ-03 | Deploy and test StegaShield | Verified |
| OBJ-04 | Evaluate controlled RGB-LSB detection | Completed |
| OBJ-05 | Investigate classification errors | Completed |
| OBJ-06 | Correlate endpoint and network activity | Verified |
| OBJ-07 | Verify reproducibility and evidence integrity | Verified |

These outcomes refer to the completed laboratory tasks, not general production capabilities.

## 3. Laboratory Architecture

The evaluation used a controlled laboratory with separate endpoint, detection, and monitoring components.

| Component | Technology | Function |
|---|---|---|
| Host workstation | MacBook Air M2 | Laboratory management |
| Windows endpoint | Windows 11 ARM64 | Controlled test submissions |
| Linux server | Ubuntu ARM64 | Detection service and network sensor |
| Detection engine | StegaShield | Image classification |
| Container runtime | Docker | Application deployment |
| Reverse proxy | Nginx | HTTPS access and request logging |
| Endpoint telemetry | Sysmon | Process and network events |
| SIEM | Splunk Enterprise | Centralized Windows investigation |
| Network monitoring | Zeek | Connection metadata |
| Dataset preparation | Python and Pillow | Independent image generation |

### Verified Traffic Flow

```text
Windows 11 — WIN-SOC
192.168.64.20
        |
        | HTTPS TCP 443
        v
Ubuntu — Nginx
192.168.64.19
        |
        | Local HTTP proxy
        v
StegaShield Docker API
127.0.0.1:3000
```

Sysmon events from Windows were forwarded to Splunk Enterprise running on macOS.

Zeek independently recorded network connection metadata on Ubuntu.

Nginx recorded HTTP request information.

The experiment did not establish native ingestion of all Zeek and Nginx records into Splunk.

## 4. Completed Investigation Phases

The seven practical stages are documented separately.

| Phase | Scope | Documentation |
|---|---|---|
| 01 | Environment & Laboratory Baseline | [Phase 01](../phase-01-environment-baseline/README.md) |
| 02 | Independent Dataset & Ground Truth | [Phase 02](../phase-02-independent-dataset/README.md) |
| 03 | StegaShield Deployment & Clean Baseline | [Phase 03](../phase-03-deployment-baseline/README.md) |
| 04 | RGB-LSB Detection Validation | [Phase 04](../phase-04-lsb-detection-validation/README.md) |
| 05 | Error Analysis & Generalization Limits | [Phase 05](../phase-05-error-analysis/README.md) |
| 06 | HTTPS & SOC Telemetry Correlation | [Phase 06](../phase-06-soc-telemetry-correlation/README.md) |
| 07 | Reproducibility & Evidence Integrity | [Phase 07](../phase-07-reproducibility-integrity/README.md) |

Phase 08 consolidates their findings and does not represent an eighth experimental stage.

## 5. Independent Test Dataset

The primary detection validation used Dataset B.

### Clean Reference

| Property | Value |
|---|---|
| Sample ID | SS-001-CLEAN |
| Filename | SS-001-CLEAN.png |
| Ground truth | CLEAN |
| Dimensions | 512 × 512 |
| Mode | RGB |
| Format | PNG |
| Generator | Python and Pillow |
| Random seed | 20261007 |
| Embedded payload | None |

### Modified Reference

| Property | Value |
|---|---|
| Sample ID | SS-001-LSB-RAW |
| Filename | SS-001-LSB-RAW.png |
| Ground truth | MODIFIED |
| Parent sample | SS-001-CLEAN |
| Embedding method | RGB-LSB |
| Payload preparation | RAW |
| Payload length | 23 bytes |
| Length header | 32 bits |
| Total embedded positions | 216 |

The embedded payload was:

```text
STEGASHIELD_PILOT_SS001
```

The samples were independently generated and labeled before StegaShield evaluation.

Both samples were synthetic RGB images rather than natural photographs.

The complete dataset methodology is documented in [Phase 02](../phase-02-independent-dataset/README.md).

## 6. Detection Validation Results

### 6.1 Recorded Model Predictions

| Sample | Ground Truth | Prediction | Probability | Outcome |
|---|---|---|---|---|
| SS-001-CLEAN.png | CLEAN | clean | 0.20640554 | True Negative |
| SS-001-LSB-RAW.png | MODIFIED | clean | 0.20640557 | False Negative |

The clean sample was correctly classified.

The modified sample was incorrectly classified as clean.

### 6.2 Probability Comparison

The difference between the two reported probabilities was:

```text
0.20640557 - 0.20640554 = 0.00000003
```

The detector reported almost identical probabilities for the clean and modified images.

This observation applies to the tested image pair.

It does not explain the model's internal decision process.

### 6.3 Repeated Modified-Image Result

Two saved analysis records for the modified-image filename contained:

| Record | Prediction | Probability |
|---|---|---|
| Earlier analysis | clean | 0.20640557 |
| Later analysis | clean | 0.20640557 |

The records had separate analysis IDs and timestamps.

Both reported the same prediction.

However, the saved JSON records did not contain input-image SHA-256 fingerprints.

Therefore, the records alone do not independently establish identical uploaded bytes across both submissions.

### 6.4 Confusion Matrix — Observed Sample Pair

| Actual Condition | Predicted Clean | Predicted Modified |
|---|---:|---:|
| CLEAN | 1 | 0 |
| MODIFIED | 1 | 0 |

These are the observed counts for the two-sample experiment.

They must not be interpreted as representative estimates of model sensitivity, specificity, or overall accuracy.

### Finding

**Finding ID:** FINAL-F01

**Title:** Controlled RGB-LSB Modification Not Detected

**Ground truth:** MODIFIED

**Observed prediction:** clean

**Classification:** False Negative

**Evidence:** Independent sample generation, saved analysis response, verified image fingerprint, and successful payload recovery.

**Conclusion:** A missed detection occurred for the tested modified image.

**Generalization:** Not established.

Detailed validation evidence is documented in [Phase 04](../phase-04-lsb-detection-validation/README.md) and [Phase 05](../phase-05-error-analysis/README.md).

## 7. Independent Payload Verification

The modified image was created by inserting a controlled 23-byte message into RGB least-significant bits.

The embedding procedure used a 32-bit length header followed by the payload bits.

During the later evidence verification, I performed a read-only extraction from the preserved modified PNG.

The recorded output was:

```text
Payload length: 23
Recovered payload: STEGASHIELD_PILOT_SS001
```

This matched the intended payload defined in the embedding script.

### Interpretation

The result provided independent confirmation that the preserved modified image contained the expected embedded data.

It strengthened the false-negative finding because the modified ground truth was supported by more than the filename or manifest label.

The extraction procedure relied on knowledge of the embedding format.

It did not establish general-purpose detection or recovery of unknown steganographic payloads.

## 8. SOC Telemetry Investigation

The SOC investigation focused on authorized image submissions from WIN-SOC to the StegaShield server.

Both images were submitted over HTTPS to:

```text
https://192.168.64.19/api/analyze
```

### 8.1 Nginx Evidence

Nginx recorded two successful POST requests from:

```text
192.168.64.20
```

Both requests targeted:

```text
/api/analyze
```

Both returned HTTP 200.

This confirmed that the reverse proxy recorded successful application requests.

### 8.2 Zeek Evidence

Zeek's archived connection logs contained corresponding TCP/SSL connections.

| Submission | Source Port | Destination | Service |
|---|---:|---|---|
| Clean image | 54301 | 192.168.64.19:443 | SSL |
| Modified image | 54306 | 192.168.64.19:443 | SSL |

The source ports provided useful correlation identifiers.

Zeek recorded connection metadata, not the encrypted image content.

### 8.3 Sysmon Evidence

Sysmon Event ID 3 recorded network connections from WIN-SOC to the Ubuntu server.

The clean-image connection identified:

```text
C:\WINDOWS\system32\curl.exe
```

The modified-image connection recorded:

```text
Image: <unknown process>
ProcessId: 10344
SourcePort: 54306
```

This created an incomplete process-attribution record.

### 8.4 Splunk Process Correlation

I investigated the corresponding process creation telemetry using Sysmon Event ID 1 in Splunk.

The record identified:

```text
ProcessId: 10344
Image: C:\Windows\System32\curl.exe
ParentImage: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

The command line referenced the modified-image filename and the StegaShield HTTPS API endpoint.

The process ID, timing, executable path, parent process, and command-line context supported attribution to `curl.exe`.

### Finding

**Finding ID:** FINAL-F02

**Title:** Incomplete Network Process Attribution Reconstructed

**Initial observation:** Sysmon Event ID 3 reported an unknown process image.

**Correlation:** Sysmon Event ID 1 using process ID 10344 and supporting context.

**Attributed executable:** curl.exe

**Parent process:** powershell.exe

**Conclusion:** Strong attribution of the authorized network activity was obtained through endpoint telemetry correlation.

### Limitations

- Process IDs can be reused.
- The network event did not provide a directly matching usable process GUID.
- Complete cross-system clock synchronization was not independently established.
- The investigation did not demonstrate native ingestion of the specific Zeek and Nginx records into Splunk.
- The activity was authorized laboratory testing, not a confirmed malicious incident.

The investigation and selected screenshots are documented in [Phase 06](../phase-06-soc-telemetry-correlation/README.md).

## 9. Reproducibility and Integrity Assessment

The pilot preserved original and reproduced image artifacts, analysis responses, checksum records, and Python scripts.

### 9.1 Image Integrity

Four image checksum verifications returned `OK`.

| Artifact | Result |
|---|---|
| Original clean image | PASS |
| Reproduced clean image | PASS |
| Original modified image | PASS |
| Reproduced modified image | PASS |

**Total: 4/4 PASS**

### 9.2 Analysis Response Integrity

Three saved analysis JSON files passed their recorded SHA-256 checks.

| Artifact | Result |
|---|---|
| Clean-image analysis | PASS |
| Modified-image analysis | PASS |
| Earlier modified-image analysis | PASS |

**Total: 3/3 PASS**

### 9.3 Docker Image Identity

The locally inspected StegaShield image ID was:

```text
sha256:64a6a378550f479c572c689bdb0af284cd81971e591a72077a6e74f7a97ea0e5
```

The inspected image architecture was `arm64`.

The deployed container referenced the same image ID and was running at the time of verification.

### 9.4 Temporary Artifact Cleanup

The temporary Ubuntu file-transfer directory was removed.

The directory-absence verification returned a passing result.

Temporary Windows test images and the temporary certificate trust entry were also removed during cleanup.

The original Dataset B artifacts were preserved.

### Interpretation

These checks support integrity and reproducibility relative to the recorded manifests and tested environment.

They do not independently establish a complete timestamped forensic chain of custody, reproducibility across every software version, or access to the model's internal training artifacts.

Detailed evidence is documented in [Phase 07](../phase-07-reproducibility-integrity/README.md).

## 10. Consolidated Evidence Matrix

This matrix identifies the evidence supporting the final findings.

| ID | Evidence | Source Phase | Verification |
|---|---|---|---|
| EV-01 | Laboratory telemetry baseline | 01 | Verified through recorded tests |
| EV-02 | Clean-image generation | 02 | Verified |
| EV-03 | RGB-LSB embedding | 02 | Verified |
| EV-04 | Ground-truth manifest | 02 | Validated |
| EV-05 | StegaShield Docker deployment | 03 | Verified |
| EV-06 | Clean-image classification | 03–05 | True Negative |
| EV-07 | Modified-image classification | 04–05 | False Negative |
| EV-08 | Repeated modified-image responses | 04–05 | Matching recorded predictions |
| EV-09 | Nginx HTTPS request records | 06 | Verified |
| EV-10 | Zeek connection metadata | 06 | Verified |
| EV-11 | Sysmon network connection events | 06 | Verified |
| EV-12 | Sysmon process creation correlation | 06 | Verified |
| EV-13 | Four image checksum checks | 07 | 4/4 PASS |
| EV-14 | Three analysis JSON checksum checks | 07 | 3/3 PASS |
| EV-15 | Docker image identity | 07 | Verified |
| EV-16 | Embedded payload recovery | 07 | PASS |
| EV-17 | Temporary transfer cleanup | 07 | Verified |

### Evidence Publication Boundary

This matrix summarizes evidence reviewed during the laboratory investigation.

It does not establish that every underlying artifact has been uploaded to GitHub.

The repository's individual phase folders contain the selected screenshots and technical documentation intended for public review.

Phase 01 is documented without screenshots.

No missing image is represented as published evidence.

## 11. Consolidated Findings

### FINAL-F01 — RGB-LSB False Negative

The tested deployment incorrectly classified a controlled modified image as clean.

**Status:** Confirmed for the tested sample.

### FINAL-F02 — Process Attribution Through Correlation

Sysmon Event ID 1 supported identification of the executable associated with an Event ID 3 record containing an unknown process image.

**Status:** Verified through correlated endpoint evidence.

### FINAL-F03 — Reproducibility and Integrity

Four image artifacts and three analysis JSON records passed their recorded SHA-256 checks.

**Status:** Verified relative to the preserved manifests.

### FINAL-F04 — Independent Payload Recovery

The embedded 23-byte payload was successfully recovered from the modified PNG.

**Status:** Verified.

### FINAL-F05 — Model Generalization Remains Unknown

The experiment did not establish detection performance across a broader representative dataset.

**Status:** Documented limitation.

## 12. Research Limitations

### Small Dataset

The primary evaluation used one clean reference and one corresponding modified image.

This is insufficient for statistically meaningful generalization.

### Synthetic Carrier Image

The clean sample contained pseudorandom RGB data.

It was not representative of natural photographic images.

### Single Embedding Method

Only the controlled sequential RGB-LSB embedding method was evaluated in the primary sample pair.

### Fixed Payload Size

The modified image contained a 23-byte payload.

Other payload sizes and embedding densities were not evaluated in this experiment.

### Limited Model Visibility

The model's training data, learned features, internal decision logic, and weights were not independently audited.

The exact cause of the false-negative classification therefore remains unknown.

### Input Attribution

Saved analysis JSON responses did not include input-image SHA-256 fields.

This limits independent verification of input-byte identity across repeated submissions.

### Network Visibility

Zeek supplied connection metadata for HTTPS traffic.

It did not demonstrate inspection of the embedded image payload within encrypted traffic.

### Telemetry Integration

Windows Sysmon logs were investigated centrally in Splunk.

The project did not establish native StegaShield, Zeek, or Nginx ingestion into the same Splunk investigation pipeline.

### Clock Synchronization

Complete synchronization across Windows, Ubuntu, and Splunk timestamp representations was not independently demonstrated.

### Evidence Provenance

The checksum checks establish integrity relative to recorded manifests.

They do not constitute a complete independent forensic chain of custody.

## 13. Recommendations

The following recommendations identify future work rather than activities already completed.

### 13.1 Expand the Independent Dataset

Test a larger set of clean and modified images.

Include natural photographs, different image properties, and representative carrier content.

### 13.2 Test Additional Payload Conditions

Vary payload size, embedding density, and embedding algorithms.

Report results separately for each condition.

### 13.3 Improve Request-to-Response Traceability

Record the exact SHA-256 fingerprint of every submitted image alongside its analysis identifier and response.

### 13.4 Improve Telemetry Integration

Consider securely forwarding relevant Zeek and Nginx records into the central SIEM.

This could simplify future cross-source correlation, but it was not demonstrated in this pilot.

### 13.5 Strengthen Time Correlation

Verify clock synchronization and document timestamp representations across hosts before larger investigations.

### 13.6 Preserve Deployment Identity

Use fixed image digests and documented software versions in future experiments instead of relying only on mutable tags.

### 13.7 Use Representative Performance Metrics

Calculate aggregate detection metrics only after collecting a sufficiently large and representative test dataset.

Avoid making broad performance claims from isolated examples.

## 14. Lessons Learned

### Lesson 01 — Verify Telemetry Before Testing

The initial laboratory baseline revealed that an enabled monitoring capability did not guarantee usable network events.

Actual event generation and centralized ingestion required separate verification.

### Lesson 02 — Establish Independent Ground Truth

The false-negative finding was possible because the sample condition was known before the model produced its classification.

### Lesson 03 — Separate Availability From Detection Quality

A healthy container and successful API response did not guarantee a correct detection result.

### Lesson 04 — Correlate Multiple Evidence Sources

Sysmon process creation, network connection metadata, Zeek records, and Nginx logs provided complementary investigation context.

### Lesson 05 — Verify Important Findings Independently

Payload recovery and checksum verification strengthened the evidence supporting the controlled experiment.

### Lesson 06 — Report Limitations Transparently

A specific missed detection is a meaningful finding.

It should not be expanded into unsupported claims about overall detector performance.

## 15. Final Technical Assessment

The StegaShield Detection Validation Pilot completed its planned controlled research workflow.

The laboratory baseline was verified, an independent dataset was generated, the detector was deployed, and clean and modified samples were evaluated.

StegaShield correctly classified the known-clean sample but incorrectly classified the corresponding RGB-LSB-modified sample as clean.

This was a confirmed false negative for the tested modified image.

The embedded 23-byte payload was subsequently recovered, strengthening the independent ground-truth verification.

The SOC investigation demonstrated that endpoint, network, and application records could be correlated to reconstruct authorized HTTPS image submissions.

One important investigation finding involved an unresolved process image in Sysmon Event ID 3. Correlation with Sysmon Event ID 1 supported attribution to `curl.exe` launched from PowerShell.

The integrity review also verified the preserved image and analysis artifacts against their recorded SHA-256 checksums.

### Overall Conclusion

The experiment demonstrated a specific detection failure within a controlled laboratory.

It did not establish StegaShield's overall reliability, broader accuracy, or performance against other steganographic conditions.

The main value of the project was the evidence-driven approach:

**Establish ground truth. Generate controlled activity. Verify telemetry. Investigate unexpected results. Correlate evidence. Reproduce findings. Report limitations.**

This approach provides a defensible foundation for further SOC investigation and detection-validation research.

**Phase 08 Status: Consolidated Technical Assessment Completed.**

**Overall Project Status: Seven Laboratory Phases Documented and Consolidated.**

**Primary Detection Finding: One True Negative and One False Negative.**

**Broader Model Performance: Not Established.**

---

[Back to Main Project](../README.md)
