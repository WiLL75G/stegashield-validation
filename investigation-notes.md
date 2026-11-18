
# StegaShield Detection Validation Pilot

## Investigation Notes — SOC Analyst Technical Handoff

**Project:** StegaShield Detection Validation Pilot  
**Investigation Type:** Independent Detection Validation and SOC Telemetry Correlation  
**Environment:** Controlled SOC Home Laboratory  
**Investigation Period:** November 2026  
**Status:** Completed  
**Scope:** Seven hands-on investigation stages  

[Back to Project Overview](README.md)

---

# 1. Executive Investigation Summary

This document provides a detailed technical handoff of the StegaShield Detection Validation Pilot.

The investigation evaluated how StegaShield classified independently generated clean and steganographically modified images.

It also examined whether the associated image-submission activity could be reconstructed using endpoint, network, and application telemetry.

The objective was not simply to deploy a detection tool. The investigation focused on establishing independent ground truth, validating model responses, investigating unexpected classifications, correlating security telemetry, and verifying the integrity of retained evidence.

The hands-on investigation was completed in seven stages:

1. Laboratory environment and telemetry baseline
2. Independent dataset generation
3. StegaShield deployment and clean baseline
4. RGB-LSB detection validation
5. Detection error analysis
6. SOC telemetry correlation
7. Reproducibility and evidence integrity

An eighth GitHub documentation phase consolidates the technical assessment without introducing additional experiments.

## 1.1 Primary Detection Finding

StegaShield correctly classified an independently generated clean image but incorrectly classified its corresponding RGB-LSB-modified image as clean.

The modified image contained the controlled 23-byte payload:

```text
STEGASHIELD_PILOT_SS001
```

The payload was subsequently recovered directly from the preserved modified PNG.

This independently confirmed the presence of embedded information in the tested image.

**Detection disposition: One true negative and one confirmed false negative for the controlled image pair.**

## 1.2 Primary SOC Investigation Finding

During the controlled HTTPS image submissions, a Sysmon Event ID 3 network connection associated with the modified image recorded:

```text
Image: <unknown process>
ProcessId: 10344
SourcePort: 54306
```

Correlation with Sysmon Event ID 1 in Splunk identified the process as `curl.exe`, launched from PowerShell.

This demonstrated the value of process creation telemetry when network connection records contain incomplete process attribution.

## 1.3 Overall Assessment

The investigation demonstrated:

- Successful deployment and baseline verification.
- Independent generation of labeled test samples.
- One confirmed false-negative classification.
- Consistent recorded predictions for the modified-image filename.
- Endpoint, network, and application telemetry correlation.
- Successful recovery of the embedded payload.
- Passing SHA-256 verification of preserved evidence.

These results apply to the tested environment and samples.

They do not establish StegaShield's overall accuracy or performance against other steganographic techniques.

---

# 2. Investigation Scope

## 2.1 Objectives

The investigation addressed the following technical objectives:

1. Establish a working endpoint and network monitoring baseline.
2. Generate independently labeled clean and modified image samples.
3. Deploy and verify StegaShield on Ubuntu ARM64.
4. Compare model predictions against independent ground truth.
5. Investigate classification errors and probability differences.
6. Correlate controlled HTTPS submissions using SOC telemetry.
7. Verify artifact integrity and reproducibility.

## 2.2 Authorization

All activities were performed within an authorized laboratory.

The work included:

- Controlled image generation and modification.
- Authorized API submissions.
- Windows endpoint monitoring.
- Linux network log inspection.
- Temporary certificate-trust configuration.
- Evidence collection and integrity checks.
- Temporary artifact cleanup.

No unauthorized systems were targeted.

No production compromise or malicious data exfiltration was demonstrated.

## 2.3 Exclusions

The investigation did not independently establish:

- General model accuracy or recall.
- Production deployment reliability.
- Detection of all steganography algorithms.
- Internal model weights or training data.
- Native ingestion of every telemetry source into Splunk.
- Full visibility into encrypted HTTPS contents.
- Complete forensic chain of custody.

---

# 3. Laboratory Environment

## 3.1 Components

| Component | Technology | Function |
|---|---|---|
| Host workstation | Mac M2 | Laboratory management and Splunk Enterprise |
| Windows endpoint | Windows 11 ARM64 — WIN-SOC | Controlled image submissions |
| Linux server | Ubuntu 24.04 ARM64 | Detection service and network monitoring |
| Endpoint monitoring | Sysmon | Process and network events |
| Log forwarding | Splunk Universal Forwarder | Windows event forwarding |
| SIEM | Splunk Enterprise | Windows event investigation |
| Network monitoring | Zeek | Network connection metadata |
| Application proxy | Nginx | HTTPS requests and access logs |
| Detection engine | StegaShield | Image steganography classification |
| Dataset tools | Python and Pillow | Image generation and verification |
| Container runtime | Docker | StegaShield deployment |

## 3.2 Verified Network Addresses

| System | Address |
|---|---|
| WIN-SOC | 192.168.64.20 |
| Ubuntu server | 192.168.64.19 |
| HTTPS endpoint | 192.168.64.19:443 |
| StegaShield local API | 127.0.0.1:3000 |

## 3.3 Application Traffic Flow

```text
Windows 11 — WIN-SOC
192.168.64.20
        |
        | HTTPS TCP 443
        v
Ubuntu — Nginx
192.168.64.19
        |
        | Local HTTP reverse proxy
        v
StegaShield Docker API
127.0.0.1:3000
```

Windows Sysmon events were forwarded to Splunk Enterprise on macOS.

Zeek independently recorded network connection activity on Ubuntu.

Nginx recorded application requests.

The telemetry sources were correlated analytically. The investigation did not establish native centralization of all specific Zeek and Nginx records into Splunk.

---

# 4. Stage 1 — Laboratory Preparation and Telemetry Baseline

**Supporting documentation:** [Phase 01 — Environment Baseline](phase-01-environment-baseline/README.md)

## 4.1 Objective

Verify that the laboratory monitoring environment could collect the endpoint and network activity required for subsequent testing.

## 4.2 Initial Environment

The Windows endpoint used Sysmon for event generation and the Splunk Universal Forwarder for centralized event collection.

Ubuntu used Zeek for network monitoring.

Splunk Enterprise ran on the Mac host.

## 4.3 Initial Observation

Sysmon Event ID 3 was not appearing as expected.

Event ID 3 records network connection activity when enabled and successfully collected.

The absence of expected events created a visibility gap.

## 4.4 Investigation

The troubleshooting process included inspection of:

- Sysmon installation and service state.
- Sysmon configuration.
- Relevant event collection rules.
- Sysmon driver behavior.
- Windows event channels.
- Supporting collection errors.
- Splunk forwarding configuration.

The Sysmon configuration was updated to include the required event types.

These included:

- Event ID 1 — Process creation.
- Event ID 3 — Network connection.
- Event ID 5 — Process termination.

## 4.5 Additional Forwarding Issue

The Splunk Universal Forwarder encountered an access problem reading the Sysmon Operational event channel.

The investigation included reviewing service permissions and event-channel access.

The relevant Event Log Readers membership was updated.

The forwarder was restarted.

## 4.6 Verification

After the changes, Sysmon Event ID 3 was verified locally and in Splunk.

A recorded 30-minute search returned:

| Event ID | Description | Event Count |
|---|---|---:|
| 1 | Process creation | 215 |
| 3 | Network connection | 52 |
| 5 | Process termination | 217 |
| 16 | Configuration state change | 1 |
| **Total** | | **485** |

These counts apply only to the observed 30-minute search window.

They are not lifetime event totals or proof of general monitoring accuracy.

## 4.7 Zeek Baseline

Zeek was verified in standalone operation on Ubuntu.

The available evidence included network connection records and controlled HTTP activity.

An HTTP request returned:

```text
204 No Content
```

This supported the observed HTTP monitoring functionality.

It did not establish visibility into every virtual-machine network path or decrypted HTTPS contents.

## 4.8 Finding S1-F01 — Missing Network Telemetry

**Observation:** Expected Sysmon Event ID 3 records were initially absent.

**Investigation:** Reviewed configuration, event generation, and supporting collection behavior.

**Remediation:** Updated event rules and verified collection.

**Disposition:** Event ID 3 subsequently appeared locally and in Splunk.

**Limitation:** A single definitive root cause for the initial local event absence was not established.

## 4.9 Finding S1-F02 — Event Channel Access

**Observation:** The forwarding service encountered a Sysmon Operational channel access problem.

**Remediation:** Updated relevant Event Log Readers membership and restarted the forwarder.

**Disposition:** Required Windows telemetry subsequently became searchable in Splunk.

## 4.10 Analyst Assessment

The laboratory monitoring baseline was suitable for the subsequent controlled investigation after the required events were verified.

The main operational lesson was that monitoring configuration must be validated through actual event generation and collection.

---

# 5. Stage 2 — Independent Dataset Generation

**Supporting documentation:** [Phase 02 — Independent Dataset](phase-02-independent-dataset/README.md)

## 5.1 Objective

Create independently labeled image samples without relying on the detector's predictions to establish ground truth.

## 5.2 Clean Reference

The clean reference was generated using Python and Pillow.

### Recorded Properties

| Property | Value |
|---|---|
| Sample ID | SS-001-CLEAN |
| Filename | SS-001-CLEAN.png |
| Ground truth | CLEAN |
| Dimensions | 512 × 512 |
| Image mode | RGB |
| Format | PNG |
| Random seed | 20261007 |
| Embedded payload | None |

The sample consisted of pseudorandom RGB pixel values.

It was not a natural photographic image.

The generation procedure used a fixed random seed to support deterministic reproduction.

## 5.3 Modified Reference

The modified image was derived from the clean sample using RGB least-significant-bit replacement.

### Recorded Properties

| Property | Value |
|---|---|
| Sample ID | SS-001-LSB-RAW |
| Filename | SS-001-LSB-RAW.png |
| Ground truth | MODIFIED |
| Parent | SS-001-CLEAN.png |
| Embedding method | RGB-LSB |
| Payload preparation | RAW |
| Payload length | 23 bytes |

## 5.4 Embedded Payload

```text
STEGASHIELD_PILOT_SS001
```

The embedding procedure used:

- A 32-bit payload-length header.
- 184 payload bits.
- 216 total embedded bit positions.

The central Python operation was:

```python
channels[index] = (channels[index] & 254) | int(bit)
```

This operation clears the least significant bit of a selected RGB channel byte and replaces it with the intended bit.

The technique modified image-channel values rather than appending text to the end of the PNG.

## 5.5 Ground-Truth Manifest

The ground-truth CSV recorded:

- Sample identifiers.
- Ground-truth labels.
- Parent-child image relationship.
- File paths.
- Dimensions.
- Image format and mode.
- Embedding method.
- Payload preparation.
- Payload size.
- SHA-256 fingerprints.

The initial manifest structure was malformed and did not parse correctly.

After correction, the validation output confirmed:

```text
Columns: 12
Records: 2
```

Both expected sample identifiers were present.

The original-image hash checks passed.

## 5.6 Original Image Fingerprints

### Clean Image

```text
d6fc09cec7ccd77d509457f610ab1d3f73a9e68dc3bbc146cb383e5255f34d5a
```

### Modified Image

```text
a13757e106766d7d4960c151158c56131245646ef4cb50f28208649d4db97570
```

## 5.7 Evidence Preservation

The original images were preserved separately.

Reproduction scripts wrote output into distinct evidence paths.

Exclusive file creation was used to reduce the risk of accidentally overwriting preserved artifacts.

## 5.8 Finding S2-F01 — Independent Ground Truth

**Observation:** One clean and one RGB-LSB-modified sample were generated.

**Verification:** Ground-truth manifest and recorded image fingerprints.

**Disposition:** Samples suitable for the planned controlled validation.

## 5.9 Limitations

The dataset contained only one synthetic clean image and one modified derivative.

It did not represent natural photographs or multiple embedding methods.

No representative benchmark was established.

---

# 6. Stage 3 — StegaShield Deployment and Clean Baseline

**Supporting documentation:** [Phase 03 — Deployment Baseline](phase-03-deployment-baseline/README.md)

## 6.1 Objective

Deploy StegaShield on Ubuntu ARM64, confirm application readiness, and establish a known-clean classification baseline.

## 6.2 Docker Deployment

The recorded image reference was:

```text
5iprojects/stegashield:latest
```

### Relevant Configuration

| Setting | Value |
|---|---|
| Container | stegashield |
| Platform | linux/arm64 |
| Host binding | 127.0.0.1:3000 |
| Container port | 3000 |
| Maximum upload size | 10,485,760 bytes |
| Maximum image dimension | 4,000 |
| Inference timeout | 60 seconds |
| Restart policy | no |

The deployment required a license token supplied through the local environment.

The token is not included in the public documentation.

## 6.3 Application Readiness

The local readiness endpoint was:

```text
http://127.0.0.1:3000/ready
```

The endpoint returned:

```text
HTTP 200
```

The container reported a healthy running state.

This verified application availability for the controlled test.

It did not establish the model's detection accuracy.

## 6.4 Clean Reference Analysis

The known-clean image was submitted to the detection engine.

The preserved response was:

```json
{
  "id": "35924038-6e0b-40df-90ff-4ee501c770f8",
  "filename": "SS-001-CLEAN.png",
  "stego_probability": 0.20640554,
  "label": "clean",
  "created_at": "2026-11-17T19:42:37.944816672+00:00"
}
```

### Evaluation

| Field | Value |
|---|---|
| Ground truth | CLEAN |
| Model prediction | clean |
| Probability | 0.20640554 |
| Outcome | True Negative |

The clean reference was classified correctly.

## 6.5 HTTPS Configuration

Nginx was configured as an HTTPS reverse proxy.

The configuration included:

- TCP port 443.
- TLS 1.2 and TLS 1.3.
- A self-signed laboratory certificate.
- Client IP allowlisting.
- Reverse proxying to the local StegaShield API.
- Separate application access and error logs.

The initial Windows connection failed certificate validation.

The observed error included:

```text
SEC_E_UNTRUSTED_ROOT
```

The certificate-trust issue was addressed during the later controlled HTTPS workflow.

## 6.6 Finding S3-F01 — Application Readiness

**Observation:** Container healthy and readiness endpoint returned HTTP 200.

**Disposition:** Verified.

## 6.7 Finding S3-F02 — Clean Baseline

**Observation:** Known-clean sample classified as clean.

**Disposition:** True Negative for the tested image.

## 6.8 Limitations

A healthy container and successful API response do not establish overall detection performance.

The clean baseline represented one synthetic sample.

---

# 7. Stage 4 — RGB-LSB Detection Validation

**Supporting documentation:** [Phase 04 — RGB-LSB Detection](phase-04-lsb-detection-validation/README.md)

## 7.1 Objective

Determine whether StegaShield identified the independently generated RGB-LSB-modified image.

## 7.2 Test Sample

```text
Filename: SS-001-LSB-RAW.png
Ground truth: MODIFIED
Embedding method: RGB-LSB
Payload length: 23 bytes
```

The embedded message was:

```text
STEGASHIELD_PILOT_SS001
```

## 7.3 Recorded Analysis

The saved StegaShield response was:

```json
{
  "id": "af3bfda8-b590-4617-a805-bac6f09dce29",
  "filename": "SS-001-LSB-RAW.png",
  "stego_probability": 0.20640557,
  "label": "clean",
  "created_at": "2026-11-17T20:05:19.979548339+00:00"
}
```

## 7.4 Expected Versus Observed

| Field | Value |
|---|---|
| Ground truth | MODIFIED |
| Prediction | clean |
| Probability | 0.20640557 |
| Classification | False Negative |

The application processed the modified image and returned a valid response.

However, the predicted label did not match the independently established ground truth.

## 7.5 Finding S4-F01 — RGB-LSB False Negative

**Observation:** Modified PNG classified as clean.

**Supporting evidence:**

- Independent embedding methodology.
- Ground-truth manifest.
- Preserved modified PNG.
- StegaShield analysis JSON.
- Successful payload recovery.
- SHA-256 verification.

**Disposition:** Confirmed False Negative for the tested sample.

## 7.6 Interpretation

The tested detector missed this particular controlled RGB-LSB modification.

The result does not demonstrate that all RGB-LSB images evade detection.

The internal cause of the incorrect classification was not established.

---

# 8. Stage 5 — Detection Error Analysis

**Supporting documentation:** [Phase 05 — Error Analysis](phase-05-error-analysis/README.md)

## 8.1 Objective

Compare the recorded model responses and determine what could be concluded from the observed classification error.

## 8.2 Prediction Comparison

| Sample | Ground Truth | Prediction | Probability |
|---|---|---|---|
| SS-001-CLEAN.png | CLEAN | clean | 0.20640554 |
| SS-001-LSB-RAW.png | MODIFIED | clean | 0.20640557 |

The reported probability difference was:

```text
0.20640557 - 0.20640554 = 0.00000003
```

The two samples received almost identical reported probabilities.

## 8.3 Repeated Modified-Image Analysis

An earlier saved response contained:

```json
{
  "id": "385dd15d-b387-4c6a-990e-6c153ffc3d3b",
  "filename": "SS-001-LSB-RAW.png",
  "stego_probability": 0.20640557,
  "label": "clean",
  "created_at": "2026-11-17T20:01:39.687291170+00:00"
}
```

The two modified-image records contained:

| Record | Prediction | Probability |
|---|---|---|
| Earlier analysis | clean | 0.20640557 |
| Later analysis | clean | 0.20640557 |

They had different analysis IDs and timestamps.

Both reported the same label and probability.

## 8.4 Interpretation

The recorded responses support consistent reported predictions across the two saved modified-image analyses.

However, the JSON responses did not include input-image SHA-256 fields.

Matching filenames and probabilities alone do not independently prove identical uploaded bytes across both submissions.

## 8.5 Observed Classification Counts

| Actual Condition | Predicted Clean | Predicted Modified |
|---|---:|---:|
| CLEAN | 1 | 0 |
| MODIFIED | 1 | 0 |

These counts describe only the controlled sample pair.

They must not be used to make population-level accuracy or false-negative-rate claims.

## 8.6 Hypotheses and Unresolved Questions

The observed score similarity raised possible questions about:

- Embedding-method sensitivity.
- Small payload size.
- Synthetic carrier-image content.
- Application preprocessing.
- Model training representation.

These are hypotheses and possible research directions.

The experiment did not establish any of them as the root cause.

## 8.7 Finding S5-F01 — Minimal Score Separation

**Observation:** Difference of 0.00000003.

**Disposition:** Verified observation.

**Root cause:** Not established.

## 8.8 Finding S5-F02 — Repeated Recorded Predictions

**Observation:** Two modified-image analysis records returned the same label and probability.

**Disposition:** Consistent recorded outputs.

**Limitation:** Exact uploaded input-byte identity not independently established by the saved JSON.

---

# 9. Stage 6 — SOC Telemetry Correlation

**Supporting documentation:** [Phase 06 — SOC Telemetry Correlation](phase-06-soc-telemetry-correlation/README.md)

## 9.1 Objective

Reconstruct authorized HTTPS image submissions using endpoint, network, and application evidence.

## 9.2 Controlled Activity

The Windows endpoint submitted both test images to:

```text
https://192.168.64.19/api/analyze
```

The submissions used PowerShell and `curl.exe`.

### Network Context

```text
Source: 192.168.64.20
Destination: 192.168.64.19
Destination port: 443/TCP
```

## 9.3 Nginx Investigation

The configured access log was:

```text
/var/log/nginx/stegashield-access.log
```

The relevant records showed successful requests.

| Client | Method | Endpoint | Status |
|---|---|---|---|
| WIN-SOC | POST | /api/analyze | 200 |
| WIN-SOC | POST | /api/analyze | 200 |

### Observation

Nginx recorded the expected application requests.

### Interpretation

The requests reached the reverse proxy and received HTTP 200 responses.

### Limitation

Nginx access records did not independently identify the Windows executable or image contents.

## 9.4 Zeek Network Investigation

Archived Zeek connection logs recorded corresponding encrypted connections.

| Submission | Source IP | Source Port | Destination |
|---|---|---:|---|
| Clean | 192.168.64.20 | 54301 | 192.168.64.19:443 |
| Modified | 192.168.64.20 | 54306 | 192.168.64.19:443 |

The records identified SSL connections and included `SF` connection states.

### Observation

The relevant source and destination addresses and ports were present.

### Interpretation

Source ports `54301` and `54306` provided useful correlation identifiers.

### Limitation

Zeek recorded encrypted connection metadata.

The investigation did not demonstrate decryption of the HTTPS payload.

## 9.5 Windows Sysmon Investigation

Windows Sysmon Event ID 3 captured corresponding network activity.

### Clean Submission

```text
SourcePort: 54301
DestinationIp: 192.168.64.19
DestinationPort: 443
Image: curl.exe
```

### Modified Submission

```text
SourcePort: 54306
DestinationIp: 192.168.64.19
DestinationPort: 443
ProcessId: 10344
Image: <unknown process>
```

The modified-image event contained useful network information but lacked a resolved process image.

## 9.6 Splunk Investigation

The relevant Windows events were indexed under:

```text
index=windows
host=WIN-SOC
```

### Query 1 — Clean Image Network Connection

```spl
index=windows host="WIN-SOC" EventCode=3
SourceIp="192.168.64.20"
SourcePort=54301
DestinationIp="192.168.64.19"
DestinationPort=443
| table _time host EventCode Image ProcessId SourceIp SourcePort DestinationIp DestinationPort
```

The result identified the clean-image connection and its originating executable.

### Query 2 — Modified Image Network Connection

```spl
index=windows host="WIN-SOC" EventCode=3
SourceIp="192.168.64.20"
SourcePort=54306
DestinationIp="192.168.64.19"
DestinationPort=443
| table _time host EventCode Image ProcessId SourceIp SourcePort DestinationIp DestinationPort
```

The result identified the corresponding network event containing an unknown process image.

The event retained process ID `10344`.

### Query 3 — Process Creation Investigation

```spl
index=windows host="WIN-SOC" EventCode=1 ProcessId=10344
| table _time ProcessGuid ProcessId Image ParentImage CommandLine
```

The process creation record identified:

```text
ProcessId: 10344
Image: C:\Windows\System32\curl.exe
ParentImage: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

The command line referenced the modified image and StegaShield HTTPS API endpoint.

## 9.7 Correlation Methodology

The evidence was correlated as follows:

1. Zeek recorded the modified-image HTTPS connection using source port 54306.
2. Sysmon Event ID 3 recorded the corresponding Windows network connection.
3. The network event contained PID 10344 but an unknown process image.
4. Sysmon Event ID 1 was searched using the recorded PID.
5. Event ID 1 identified `curl.exe`.
6. The parent executable was PowerShell.
7. The process command line referenced the modified PNG and API endpoint.

The process ID, connection metadata, timing, and command-line context supported attribution.

## 9.8 Correlation Caveats

Process IDs can be reused.

Therefore, PID matching alone is insufficient.

The available network event did not provide a directly matching usable process GUID.

The investigation used supporting host, timing, and command-line context.

Some timestamp representations differed across monitoring systems.

Complete clock synchronization was not independently established.

## 9.9 Findings

### S6-F01 — Application Request Visibility

**Observation:** Nginx recorded successful image-analysis POST requests.

**Disposition:** Verified.

### S6-F02 — Network Connection Visibility

**Observation:** Zeek recorded corresponding encrypted network connections.

**Disposition:** Verified.

### S6-F03 — Incomplete Process Image

**Observation:** Sysmon Event ID 3 recorded `<unknown process>` for the modified submission.

**Disposition:** Investigation required.

### S6-F04 — Process Attribution

**Observation:** Event ID 1 identified PID 10344 as `curl.exe`, launched from PowerShell.

**Disposition:** Attribution supported by correlated endpoint evidence.

## 9.10 SOC Assessment

The investigation demonstrated how authorized image-submission activity could be reconstructed using complementary telemetry.

It did not demonstrate malicious activity, unauthorized access, or exfiltration.

A controlled steganographic payload does not independently establish malicious intent.

---

# 10. Stage 7 — Reproducibility and Evidence Integrity

**Supporting documentation:** [Phase 07 — Reproducibility](phase-07-reproducibility-integrity/README.md)

## 10.1 Objective

Verify the retained image artifacts, saved analysis results, reproduction methodology, Docker deployment identity, payload recovery, and temporary artifact cleanup.

## 10.2 Image Integrity

The recorded verification command was:

```bash
sha256sum -c ~/stegashield-pilot/dataset-b-independent/evidence/SS-001-reproducibility.sha256
```

The checks covered:

- Original clean image.
- Reproduced clean image.
- Original RGB-LSB image.
- Reproduced RGB-LSB image.

All four returned `OK`.

**Result: 4/4 PASS.**

These checks established agreement with the recorded checksum manifest.

## 10.3 Analysis JSON Integrity

Three saved analysis records were verified:

```text
SS-001-CLEAN-analysis.json
SS-001-LSB-RAW-analysis.json
SS-001-LSB-RAW-earlier-analysis.json
```

All three matched their recorded SHA-256 fingerprints.

**Result: 3/3 PASS.**

### Limitation

Checksum verification does not independently establish original file creation times or a complete forensic chain of custody.

## 10.4 Reproduction Script Review

The preserved scripts included:

```text
generate_clean.py
embed_lsb.py
reproduce_clean.py
reproduce_lsb.py
```

The reproduction scripts preserved the documented generation and embedding procedures.

Outputs were written to separate paths.

Exclusive file creation was used to reduce the risk of overwriting existing output files.

## 10.5 Docker Image Identity

The recorded image reference was:

```text
5iprojects/stegashield:latest
```

The inspected image ID was:

```text
sha256:64a6a378550f479c572c689bdb0af284cd81971e591a72077a6e74f7a97ea0e5
```

The architecture was:

```text
arm64
```

The running container referenced the same image ID.

This improved traceability beyond the mutable `latest` tag.

It did not establish the model's internal weights or training data.

## 10.6 Independent Payload Recovery

A read-only Python extraction was performed against the preserved modified PNG.

The procedure decoded the 32-bit length header and reconstructed the payload from the RGB least-significant bits.

The recorded output was:

```text
Payload length: 23
Recovered payload: STEGASHIELD_PILOT_SS001
```

The recovered payload matched the original intended message.

### Interpretation

The preserved modified image genuinely contained the documented embedded information.

This independently strengthened the ground-truth evidence for the false-negative finding.

## 10.7 Temporary Artifact Cleanup

Temporary image and certificate copies had been used during controlled HTTPS testing.

The initial Ubuntu cleanup attempt did not remove every temporary file.

The remaining files were identified and removed.

The transfer directory was then removed and its absence verified.

Temporary Windows image copies and the imported test certificate trust entry were also removed.

The original dataset was preserved.

## 10.8 Findings

| Finding | Observation | Disposition |
|---|---|---|
| S7-F01 | Four image checksum checks | PASS |
| S7-F02 | Three analysis checksum checks | PASS |
| S7-F03 | Running Docker image identity | Verified |
| S7-F04 | Embedded payload recovery | PASS |
| S7-F05 | Temporary transfer cleanup | Verified |

---

# 11. Consolidated Investigation Timeline

The following table represents the logical progression of the completed laboratory investigation.

It is not a complete timestamped forensic chain of custody.

| Stage | Activity | Outcome |
|---|---|---|
| 1 | Monitoring preparation and troubleshooting | Required telemetry verified |
| 2 | Independent dataset generation | Ground truth established |
| 3 | Detector deployment and baseline | Clean image correctly classified |
| 4 | Modified-image validation | False negative observed |
| 5 | Error analysis | Minimal score separation identified |
| 6 | SOC telemetry investigation | Process attribution reconstructed |
| 7 | Integrity and reproducibility verification | Hashes passed and payload recovered |

## Recorded StegaShield Analysis Timestamps

| Analysis | Recorded Timestamp (UTC) |
|---|---|
| Clean reference | 2026-11-17T19:42:37.944816672+00:00 |
| Earlier modified analysis | 2026-11-17T20:01:39.687291170+00:00 |
| Later modified analysis | 2026-11-17T20:05:19.979548339+00:00 |

These timestamps originated from the saved API responses.

The later HTTPS telemetry investigation was a separate controlled workflow.

The analysis timestamps should not be conflated with the later Nginx access-log timestamps.

---

# 12. Evidence Matrix

| Evidence ID | Artifact or Observation | Supports |
|---|---|---|
| E01 | Sysmon and Splunk verification | Monitoring readiness |
| E02 | Ground-truth manifest | Independent sample labels |
| E03 | Clean-image generator | Clean sample provenance |
| E04 | RGB-LSB embedding script | Modification methodology |
| E05 | Original image hashes | Artifact identification |
| E06 | Docker readiness evidence | Application availability |
| E07 | Clean-image analysis JSON | True-negative result |
| E08 | Modified-image analysis JSON | False-negative result |
| E09 | Earlier modified analysis JSON | Repeated recorded prediction |
| E10 | Nginx access records | Application request activity |
| E11 | Zeek connection records | Network metadata |
| E12 | Sysmon Event ID 3 | Endpoint network activity |
| E13 | Sysmon Event ID 1 | Process attribution |
| E14 | Reproduction scripts | Documented methodology |
| E15 | Four image checksum results | Image integrity |
| E16 | Three analysis checksum results | Response integrity |
| E17 | Docker image metadata | Deployment identity |
| E18 | Payload recovery output | Independent payload verification |
| E19 | Temporary cleanup verification | Cleanup evidence |

These entries identify evidence observed during the investigation.

They do not imply that every underlying artifact is publicly available.

Selected screenshots and detailed investigation records are maintained in the individual phase documentation.

---

# 13. Consolidated Findings

## F01 — Controlled RGB-LSB False Negative

**Category:** Detection validation.

**Severity:** Not assigned.

**Observation:** StegaShield classified an independently modified RGB-LSB image as clean.

**Ground truth:** MODIFIED.

**Prediction:** clean.

**Probability:** 0.20640557.

**Supporting evidence:** Ground-truth manifest, embedding script, preserved image, model response, and successful payload recovery.

**Disposition:** Confirmed for the tested sample.

**Generalization:** Unknown.

## F02 — Minimal Probability Separation

**Category:** Detection error analysis.

**Observation:** Clean and modified probabilities differed by 0.00000003.

**Disposition:** Verified observation.

**Root cause:** Not established.

## F03 — Incomplete Network Process Attribution

**Category:** SOC telemetry investigation.

**Observation:** Event ID 3 contained `<unknown process>`.

**Supporting context:** Source port 54306 and PID 10344.

**Correlation:** Event ID 1 identified `curl.exe` launched from PowerShell.

**Disposition:** Process attribution supported by correlated evidence.

## F04 — Evidence Integrity

**Category:** Reproducibility.

**Observation:** Four image checksum checks and three analysis JSON checksum checks passed.

**Disposition:** Verified relative to recorded fingerprints.

## F05 — Independent Payload Recovery

**Category:** Ground-truth validation.

**Observation:** The original 23-byte payload was recovered from the modified PNG.

**Disposition:** Verified.

---

# 14. Research Limitations

## 14.1 Small Dataset

The evaluation involved one clean image and one corresponding modified image.

This is insufficient for representative model-performance conclusions.

## 14.2 Synthetic Image Content

The clean carrier contained pseudorandom RGB pixels.

The experiment did not establish performance on natural photographs.

## 14.3 Single Embedding Method

Only the controlled RGB-LSB embedding method was evaluated in the primary sample pair.

## 14.4 Fixed Payload

The tested payload was 23 bytes.

Different payload sizes and embedding densities were not evaluated.

## 14.5 Limited Model Internals

The model's internal weights, training data, and preprocessing behavior were not independently audited.

The exact cause of the false negative remains unknown.

## 14.6 Input Attribution

Saved analysis JSON files did not contain submitted-image SHA-256 fingerprints.

This limited independent byte-level attribution across repeated submissions.

## 14.7 Encrypted Network Visibility

Zeek observed network connection metadata.

It did not independently recover the embedded image contents from HTTPS traffic.

## 14.8 Telemetry Centralization

Windows Sysmon events were investigated in Splunk.

Native ingestion of the specific Nginx and Zeek records into Splunk was not established.

## 14.9 Timestamp Correlation

Complete cross-system clock synchronization was not independently demonstrated.

## 14.10 Evidence Provenance

Checksum comparisons established agreement with recorded fingerprints.

They did not establish a complete independent forensic chain of custody.

---

# 15. Technical Recommendations

The following items are proposed future research activities.

They were not completed during the original pilot.

## 15.1 Expand Independent Dataset Coverage

Evaluate a larger number of independently labeled clean and modified images.

Include natural photographic content and different image properties.

## 15.2 Vary Payload Conditions

Evaluate different:

- Payload sizes.
- Embedding densities.
- Carrier image types.
- Embedding methods.

## 15.3 Improve Input-to-Response Attribution

Record the exact SHA-256 fingerprint of each submitted image alongside the resulting analysis identifier.

This would improve repeatability and evidence traceability.

## 15.4 Strengthen Detection Evaluation

Use a predefined evaluation methodology and a sufficiently representative dataset before calculating aggregate accuracy, precision, recall, or false-negative-rate estimates.

## 15.5 Improve Telemetry Correlation

Consider structured application logging and centralized ingestion of relevant Nginx and Zeek records into the SIEM.

## 15.6 Improve Time Synchronization

Verify and document clock synchronization across Windows, Ubuntu, and the SIEM.

## 15.7 Preserve Deployment Identity

Record exact container image digests and relevant application versions.

Avoid depending only on mutable image tags.

---

# 16. Final Analyst Handoff

## Confirmed

- StegaShield deployed successfully on Ubuntu ARM64.
- Application readiness verified.
- Known-clean image correctly classified.
- Controlled RGB-LSB-modified image incorrectly classified as clean.
- Two saved modified-image analyses reported matching predictions.
- Nginx recorded successful authorized API requests.
- Zeek recorded corresponding encrypted network connections.
- Windows Sysmon events were searchable in Splunk.
- Sysmon Event ID 1 supported process attribution for an incomplete Event ID 3 record.
- Four image SHA-256 checks passed.
- Three analysis JSON checksum checks passed.
- Running Docker container image identity verified.
- Original 23-byte embedded payload recovered.
- Temporary transfer cleanup verified.

## Not Established

- Overall StegaShield accuracy.
- General false-negative rate.
- Detection performance for other embedding methods.
- Generalization to natural photographs.
- Internal root cause of the missed detection.
- Native StegaShield-to-Splunk integration.
- Decrypted network inspection of image contents.
- Complete cross-system time synchronization.
- Complete independent forensic chain of custody.

## Final Disposition

**Investigation status:** Completed.

**Primary detection result:** One confirmed false negative for a controlled RGB-LSB-modified sample.

**Primary SOC finding:** Process attribution reconstructed using Sysmon Event IDs 3 and 1.

**Evidence verification:** Four image checks and three analysis JSON checks passed.

**Ground-truth confirmation:** Original 23-byte payload successfully recovered.

**Overall assessment:** The investigation demonstrated a specific missed detection within a controlled laboratory and a reproducible, evidence-driven SOC investigation workflow.

The findings should not be generalized beyond the tested conditions without additional representative validation.

---

# 17. Supporting Documentation

| Phase | Technical Documentation |
|---|---|
| 01 | [Environment Baseline](phase-01-environment-baseline/README.md) |
| 02 | [Independent Dataset](phase-02-independent-dataset/README.md) |
| 03 | [Deployment Baseline](phase-03-deployment-baseline/README.md) |
| 04 | [RGB-LSB Detection Validation](phase-04-lsb-detection-validation/README.md) |
| 05 | [Error Analysis](phase-05-error-analysis/README.md) |
| 06 | [SOC Telemetry Correlation](phase-06-soc-telemetry-correlation/README.md) |
| 07 | [Reproducibility & Integrity](phase-07-reproducibility-integrity/README.md) |
| 08 | [Final Technical Assessment](phase-08-final-assessment/README.md) |

---

**End of Technical Investigation Handoff**

[Back to Project Overview](README.md)

