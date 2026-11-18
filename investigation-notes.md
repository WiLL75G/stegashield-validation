
# StegaShield Detection Validation Pilot

## Investigation Notes — Analyst Handoff & Technical Review

**Project:** StegaShield Detection Validation Pilot  
**Investigator:** William  
**Environment:** Controlled SOC Home Laboratory  
**Investigation period:** November 2026  
**Technical review:** November 20, 2026  
**Review audience:** Franco Garcia and technical stakeholders  
**Investigation type:** Independent Detection Validation and SOC Telemetry Correlation  
**Status:** Seven hands-on stages completed  

**Repository:** [StegaShield Detection Validation Pilot](https://github.com/WiLL75G/stegashield-validation)

---

# 1. Investigation Executive Summary

I conducted an independent validation pilot to evaluate how StegaShield behaved when analyzing a known-clean image and a corresponding image containing a controlled steganographic payload.

I structured the practical investigation into seven stages:

1. Laboratory environment and telemetry baseline
2. Independent dataset generation
3. StegaShield deployment and clean baseline
4. RGB-LSB detection validation
5. Detection error analysis
6. SOC telemetry correlation
7. Reproducibility and evidence integrity

The project is documented in eight GitHub phases, with Phase 08 serving as the final consolidated assessment rather than an additional experiment.

My objective was not simply to install StegaShield. I wanted to establish independent ground truth, evaluate the detector's predictions, investigate unexpected results, correlate security telemetry, and verify the preserved evidence.

## Primary Detection Finding

StegaShield correctly classified a known-clean image but incorrectly classified its corresponding RGB-LSB-modified image as clean.

The modified image contained a controlled 23-byte payload:

```text
STEGASHIELD_PILOT_SS001
```

I subsequently recovered the exact payload from the preserved modified PNG.

**Detection outcome: One true negative and one confirmed false negative for the controlled image pair.**

## Secondary SOC Finding

During the HTTPS investigation, a Sysmon Event ID 3 network connection associated with the modified-image submission recorded:

```text
Image: <unknown process>
ProcessId: 10344
```

By correlating the process ID with Sysmon Event ID 1 in Splunk, I identified the process as `curl.exe`, launched from PowerShell.

This demonstrated how endpoint event correlation can recover useful attribution when an individual network event contains incomplete information.

## Overall Assessment

The project demonstrated a specific missed detection and a practical cross-source SOC investigation.

It did not establish StegaShield's overall accuracy, production reliability, or performance against other embedding techniques.

---

# 2. Investigation Scope

## Research Questions

The investigation focused on these questions:

1. Could I independently establish clean and modified image ground truth?
2. Could StegaShield operate successfully on Ubuntu ARM64?
3. Would the model correctly classify a known-clean image?
4. Would it detect a controlled RGB-LSB modification?
5. How would the reported probabilities compare?
6. Could I independently recover the hidden payload?
7. Could I reconstruct image-submission activity through SOC telemetry?
8. Could the preserved artifacts be verified and reproduced?

## Authorization

All actions were performed within my controlled laboratory.

The image submissions, certificate testing, endpoint monitoring, and evidence collection were authorized.

No production systems were targeted.

No malicious exfiltration, unauthorized access, or real compromise was demonstrated.

## Out of Scope

The pilot did not independently establish:

- Overall detector accuracy or recall
- The model's internal architecture or training data
- Performance against all steganography methods
- Production deployment reliability
- Native integration of all telemetry sources into Splunk
- Complete forensic chain of custody

---

# 3. Laboratory Architecture

The project used a Mac M2 host with separate ARM64 virtual machines.

| Component | Role |
|---|---|
| Mac M2 | Laboratory host and Splunk Enterprise |
| Windows 11 ARM64 — WIN-SOC | Controlled image submissions |
| Ubuntu 24.04 ARM64 | StegaShield, Docker, Nginx, and Zeek |
| Sysmon | Windows process and network events |
| Splunk Universal Forwarder | Windows event forwarding |
| Splunk Enterprise | Centralized Windows event investigation |
| Zeek | Network connection metadata |
| Nginx | HTTPS reverse proxy and request logging |
| StegaShield | Image steganography classification |
| Python and Pillow | Independent image generation |

## Verified Network Addresses

| System | Address |
|---|---|
| WIN-SOC | 192.168.64.20 |
| Ubuntu server | 192.168.64.19 |
| Nginx HTTPS endpoint | 192.168.64.19:443 |
| Local StegaShield API | 127.0.0.1:3000 |

## Traffic Flow

```text
Windows 11 — WIN-SOC
192.168.64.20
        |
        | HTTPS TCP 443
        v
Ubuntu — Nginx
192.168.64.19
        |
        | HTTP reverse proxy
        v
StegaShield Docker API
127.0.0.1:3000
```

Windows Sysmon events were forwarded separately to Splunk on macOS.

Zeek recorded network connection activity on Ubuntu.

Nginx recorded HTTP application requests.

These sources were investigated together, but the experiment did not establish native centralization of all Zeek and Nginx records into Splunk.

---

# 4. Stage 1 — Laboratory Preparation & Telemetry Baseline

[View Phase 01](phase-01-environment-baseline/README.md)

## Objective

Before testing StegaShield, I wanted to verify that my SOC laboratory could reliably capture the endpoint and network activity required for the investigation.

## What I Did

I examined:

- Windows Sysmon installation and configuration
- Sysmon event generation
- Splunk Universal Forwarder permissions
- Windows event ingestion into Splunk
- Ubuntu Zeek network monitoring
- Relevant laboratory connectivity and service checks

## Initial Problem

Sysmon Event ID 3 was not appearing as expected.

Event ID 3 is used to record network connection activity when enabled and successfully collected.

I investigated the configuration, relevant event rules, event channel, service behavior, and supporting driver information.

I updated the collection configuration to include the required Event IDs 1, 3, and 5.

## Second Problem — Forwarder Access

The Splunk Universal Forwarder encountered an access issue while reading the Sysmon Operational channel.

I investigated the service permissions, updated the necessary Event Log Readers membership, and restarted the forwarder.

## Verified Result

After remediation, I verified that Sysmon Event ID 3 was generated locally and that relevant events appeared in Splunk.

A recorded 30-minute Splunk search returned:

| Event ID | Description | Events |
|---|---|---:|
| 1 | Process creation | 215 |
| 3 | Network connection | 52 |
| 5 | Process termination | 217 |
| 16 | Configuration state change | 1 |
| **Total** | | **485** |

These counts reflect one 30-minute search window, not lifetime monitoring totals.

## Zeek Verification

I also confirmed standalone Zeek monitoring on Ubuntu.

The evidence included network connection records and controlled HTTP activity.

An HTTP test returned `204 No Content`.

This supported the observed monitoring capability but did not establish decrypted HTTPS inspection or complete visibility into every virtual-machine connection.

## Finding S1-F01

**Observation:** Expected Sysmon network telemetry was initially missing.

**Investigation:** Reviewed collection rules, event generation, and forwarding.

**Remediation:** Updated relevant settings and forwarder access.

**Disposition:** Event ID 3 subsequently verified in Splunk.

**Limitation:** A single definitive root cause for the initial absence of local Event ID 3 records was not established.

## Analyst Takeaway

A monitoring tool being installed or enabled does not guarantee that the required events are being collected.

I verified the telemetry before relying on it.

### How I Explain This to Franco

"Before testing StegaShield, I checked whether my monitoring environment was actually recording the events I needed. I discovered that Sysmon network events were missing, investigated the problem, and verified them in Splunk before proceeding."

---

# 5. Stage 2 — Independent Dataset Generation

[View Phase 02](phase-02-independent-dataset/README.md)

## Objective

Create independently labeled clean and modified images so that StegaShield predictions could be evaluated against known ground truth.

## Why Independent Ground Truth Matters

If I submitted an unknown image and the model labeled it clean, I would not necessarily know whether that prediction was correct.

I therefore generated the samples myself.

## Sample A — Clean Reference

The clean image was created using Python and Pillow.

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

The image consisted of pseudorandom RGB pixels rather than natural photographic content.

The fixed seed supported deterministic reproduction under the tested generation procedure.

## Sample B — Modified Reference

I created a second image from the clean sample using RGB least-significant-bit replacement.

| Property | Value |
|---|---|
| Sample ID | SS-001-LSB-RAW |
| Filename | SS-001-LSB-RAW.png |
| Ground truth | MODIFIED |
| Parent | SS-001-CLEAN.png |
| Embedding method | RGB-LSB |
| Payload preparation | RAW |
| Payload size | 23 bytes |

## Embedded Payload

```text
STEGASHIELD_PILOT_SS001
```

The embedding method used:

- 32-bit payload-length header
- 184 payload bits
- 216 total embedded bit positions

The relevant Python operation was:

```python
channels[index] = (channels[index] & 254) | int(bit)
```

This clears the least significant bit of the selected RGB channel byte and replaces it with the required payload bit.

The method modified image channel values rather than appending text to the PNG file.

## Ground-Truth Manifest

I preserved the sample labels, relationships, dimensions, embedding parameters, payload size, and SHA-256 hashes in a CSV manifest.

The initial CSV structure required correction.

After fixing the structure, validation confirmed:

- 12 columns
- 2 records
- Correct sample identifiers
- Successful original-image hash checks

## Recorded SHA-256 Fingerprints

### Clean Image

```text
d6fc09cec7ccd77d509457f610ab1d3f73a9e68dc3bbc146cb383e5255f34d5a
```

### Modified Image

```text
a13757e106766d7d4960c151158c56131245646ef4cb50f28208649d4db97570
```

## Finding S2-F01

**Observation:** Independently generated clean and modified PNG samples were preserved.

**Ground truth:** One CLEAN and one MODIFIED.

**Verification:** Original artifacts matched their recorded fingerprints.

**Disposition:** Dataset suitable for the planned controlled test.

## Limitations

The dataset was small and synthetic.

It did not represent natural photographs, other payload sizes, or multiple steganography algorithms.

### How I Explain This to Franco

"I generated the clean image myself and embedded a controlled 23-byte message into a modified version. Because I already knew which image contained hidden information, I could independently check whether StegaShield's classification was correct."

---

# 6. Stage 3 — StegaShield Deployment & Clean Baseline

[View Phase 03](phase-03-deployment-baseline/README.md)

## Objective

Deploy StegaShield on Ubuntu ARM64, verify application readiness, and establish a known-clean image baseline.

## Docker Deployment

StegaShield was deployed using Docker Compose.

### Image

```text
5iprojects/stegashield:latest
```

### Recorded Configuration

| Setting | Value |
|---|---|
| Container | stegashield |
| Platform | linux/arm64 |
| Host binding | 127.0.0.1:3000 |
| Container port | 3000 |
| Maximum upload | 10,485,760 bytes |
| Maximum image dimension | 4,000 |
| Inference timeout | 60 seconds |
| Restart policy | no |

The configuration used a required license token supplied through the local environment.

The token is excluded from the public documentation.

## Readiness Verification

I inspected the running container and tested:

```text
http://127.0.0.1:3000/ready
```

The API returned HTTP 200.

The container reported a healthy state.

This established service availability for the controlled test.

## Clean Image Submission

I submitted the independently generated clean reference.

The saved result was:

```json
{
  "id": "35924038-6e0b-40df-90ff-4ee501c770f8",
  "filename": "SS-001-CLEAN.png",
  "stego_probability": 0.20640554,
  "label": "clean",
  "created_at": "2026-11-17T19:42:37.944816672+00:00"
}
```

## Interpretation

**Ground truth:** CLEAN

**Prediction:** clean

**Result:** True Negative

The detector correctly classified the tested clean reference.

## HTTPS Preparation

I configured Nginx as an HTTPS reverse proxy for Windows client testing.

The configuration included:

- TCP port 443
- TLS 1.2 and TLS 1.3
- A self-signed laboratory certificate
- IP allowlisting for WIN-SOC
- Reverse proxying to 127.0.0.1:3000
- Nginx access and error logs

The initial Windows connection failed certificate validation with:

```text
SEC_E_UNTRUSTED_ROOT
```

The laboratory certificate-trust issue was addressed during the later HTTPS investigation.

## Finding S3-F01

**Observation:** Container healthy and readiness endpoint returned HTTP 200.

**Disposition:** Deployment readiness verified.

## Finding S3-F02

**Observation:** Clean reference classified correctly.

**Disposition:** True Negative for the tested sample.

## Limitation

Successful application startup and HTTP responses do not establish overall detection accuracy.

### How I Explain This to Franco

"I deployed StegaShield using Docker, checked that the API was ready, and submitted my clean reference. It was classified correctly. That gave me a baseline before I tested the modified sample."

---

# 7. Stage 4 — RGB-LSB Detection Validation

[View Phase 04](phase-04-lsb-detection-validation/README.md)

## Objective

Evaluate whether StegaShield would identify the independently prepared RGB-LSB-modified image.

## Modified Sample

```text
Filename: SS-001-LSB-RAW.png
Ground truth: MODIFIED
Payload: STEGASHIELD_PILOT_SS001
Payload length: 23 bytes
```

## Saved StegaShield Response

```json
{
  "id": "af3bfda8-b590-4617-a805-bac6f09dce29",
  "filename": "SS-001-LSB-RAW.png",
  "stego_probability": 0.20640557,
  "label": "clean",
  "created_at": "2026-11-17T20:05:19.979548339+00:00"
}
```

## Expected Versus Observed

| Property | Value |
|---|---|
| Actual condition | MODIFIED |
| Model prediction | clean |
| Probability | 0.20640557 |
| Outcome | False Negative |

StegaShield processed the image but returned a classification that did not match its independently established ground truth.

## Finding S4-F01 — Confirmed False Negative

**Observation:** RGB-LSB-modified image classified as clean.

**Supporting evidence:**

- Independent embedding script
- Ground-truth manifest
- Preserved modified PNG
- Saved API analysis
- Subsequent payload recovery
- SHA-256 verification

**Disposition:** False Negative for the tested sample.

## Interpretation

The detector missed this specific controlled modification.

It does not follow that every RGB-LSB image would evade detection.

### How I Explain This to Franco

"This was my biggest discovery. I submitted an image that I knew contained hidden information, but StegaShield classified it as clean. I later recovered the exact hidden message from the preserved image, which strengthened the evidence supporting the false negative."

---

# 8. Stage 5 — Detection Error Analysis

[View Phase 05](phase-05-error-analysis/README.md)

## Objective

Analyze the recorded classifications and probabilities without overstating what the limited experiment could demonstrate.

## Comparison

| Sample | Ground Truth | Prediction | Probability |
|---|---|---|---|
| SS-001-CLEAN.png | CLEAN | clean | 0.20640554 |
| SS-001-LSB-RAW.png | MODIFIED | clean | 0.20640557 |

## Probability Difference

```text
0.20640557 - 0.20640554 = 0.00000003
```

The detector reported extremely similar probabilities for the two related samples.

## Earlier Modified-Image Analysis

A second saved analysis record contained:

```json
{
  "id": "385dd15d-b387-4c6a-990e-6c153ffc3d3b",
  "filename": "SS-001-LSB-RAW.png",
  "stego_probability": 0.20640557,
  "label": "clean",
  "created_at": "2026-11-17T20:01:39.687291170+00:00"
}
```

Both modified-image records reported:

```text
Prediction: clean
Probability: 0.20640557
```

The records contained different analysis IDs and timestamps.

## Observed Classification Counts

| Actual Condition | Predicted Clean | Predicted Modified |
|---|---:|---:|
| CLEAN | 1 | 0 |
| MODIFIED | 1 | 0 |

These counts apply only to the controlled sample pair.

## Hypotheses Considered

The similar scores raised possible questions about:

- Sensitivity to the embedding method
- Effects of the small payload
- Effects of synthetic carrier-image content
- Model preprocessing
- Training-data representation

These were possible research directions, not confirmed root causes.

## Finding S5-F01

**Observation:** Nearly identical scores for the clean and modified samples.

**Disposition:** Verified observation.

**Root cause:** Not established.

## Finding S5-F02

**Observation:** Two saved modified-image analyses returned matching labels and probabilities.

**Disposition:** Consistent recorded predictions.

**Limitation:** The saved responses did not include input-image SHA-256 fields proving byte-identical submissions.

## Analyst Conclusion

The experiment identified one false negative.

It did not establish representative model accuracy, recall, or a general false-negative rate.

### How I Explain This to Franco

"I compared the model probabilities and found that the difference was only 0.00000003. The repeated saved analyses returned the same prediction. However, I couldn't determine why the model missed the modification, so I documented that as an unresolved question rather than inventing a cause."

---

# 9. Stage 6 — SOC Telemetry Correlation

[View Phase 06](phase-06-soc-telemetry-correlation/README.md)

## Objective

Investigate authorized image submissions through endpoint, network, and application telemetry.

## Controlled HTTPS Testing

The Windows endpoint submitted both samples to:

```text
https://192.168.64.19/api/analyze
```

The submissions used PowerShell and `curl.exe`.

### Network Context

```text
Client: 192.168.64.20
Server: 192.168.64.19
Destination port: 443/TCP
```

## 9.1 Nginx Investigation

I inspected:

```text
/var/log/nginx/stegashield-access.log
```

Nginx recorded two successful requests.

| Client | Method | Endpoint | Status |
|---|---|---|---|
| WIN-SOC | POST | /api/analyze | 200 |
| WIN-SOC | POST | /api/analyze | 200 |

### Interpretation

Both requests reached the application reverse proxy and received successful HTTP responses.

The Nginx log alone did not identify the Windows executable or reveal the uploaded image contents.

## 9.2 Zeek Investigation

I inspected the archived Zeek connection logs.

The relevant connections were:

| Sample | Source IP | Source Port | Destination |
|---|---|---:|---|
| Clean | 192.168.64.20 | 54301 | 192.168.64.19:443 |
| Modified | 192.168.64.20 | 54306 | 192.168.64.19:443 |

The records reported SSL connections with `SF` state values.

### Interpretation

The source ports helped correlate network activity with Windows endpoint records.

Zeek observed encrypted connection metadata.

The experiment did not demonstrate decryption of HTTPS traffic or network-level extraction of the embedded payload.

## 9.3 Windows Sysmon Event ID 3

I examined the Windows network activity in Splunk.

### Clean Submission

The relevant record contained:

```text
SourcePort: 54301
DestinationIp: 192.168.64.19
DestinationPort: 443
Image: curl.exe
```

### Modified Submission

The corresponding event contained:

```text
SourcePort: 54306
DestinationIp: 192.168.64.19
DestinationPort: 443
ProcessId: 10344
Image: <unknown process>
```

The modified-image record contained network metadata but lacked a resolved process image.

## 9.4 Splunk Investigation Queries

The Windows events were indexed under:

```text
index=windows
host=WIN-SOC
```

### Query 1 — Clean Connection

```spl
index=windows host="WIN-SOC" EventCode=3
SourceIp="192.168.64.20"
SourcePort=54301
DestinationIp="192.168.64.19"
DestinationPort=443
| table _time host EventCode Image ProcessId SourceIp SourcePort DestinationIp DestinationPort
```

This search focused on the clean-image connection.

### Query 2 — Modified Connection

```spl
index=windows host="WIN-SOC" EventCode=3
SourceIp="192.168.64.20"
SourcePort=54306
DestinationIp="192.168.64.19"
DestinationPort=443
| table _time host EventCode Image ProcessId SourceIp SourcePort DestinationIp DestinationPort
```

The result exposed the unknown process image and PID 10344.

### Query 3 — Process Creation Pivot

```spl
index=windows host="WIN-SOC" EventCode=1 ProcessId=10344
| table _time ProcessGuid ProcessId Image ParentImage CommandLine
```

The result identified:

```text
ProcessId: 10344
Image: C:\Windows\System32\curl.exe
ParentImage: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

The command line referenced the modified PNG and StegaShield HTTPS endpoint.

## 9.5 Correlation Reasoning

The investigation followed this chain:

1. Zeek recorded an SSL connection from source port 54306.
2. Sysmon Event ID 3 recorded the corresponding Windows network connection.
3. The network event contained PID 10344 but an unknown process image.
4. I searched Sysmon Event ID 1 for the same PID.
5. The process creation record identified `curl.exe`.
6. The parent process was PowerShell.
7. The command line referenced the modified image and StegaShield API.

The matching process ID, endpoint information, timing, and command-line context supported attribution.

### Correlation Caveats

Process IDs can be reused.

The network event did not provide a directly matching usable process GUID.

Some timestamps were represented differently across the tools.

Complete cross-system time synchronization was not independently established.

## Findings

### S6-F01 — Nginx Request Logging

**Observation:** Successful POST requests recorded.

**Disposition:** Verified.

### S6-F02 — Zeek Network Visibility

**Observation:** Corresponding SSL connections identified.

**Disposition:** Verified.

### S6-F03 — Incomplete Process Attribution

**Observation:** Sysmon Event ID 3 showed `<unknown process>`.

**Disposition:** Investigation required.

### S6-F04 — Process Creation Correlation

**Observation:** Sysmon Event ID 1 identified PID 10344 as `curl.exe` launched by PowerShell.

**Disposition:** Attribution supported by correlated evidence.

## SOC Conclusion

The activity could be reconstructed using complementary endpoint, network, and application records.

This was authorized laboratory activity.

The investigation did not demonstrate malicious exfiltration or a production security incident.

### How I Explain This to Franco

"I investigated the image submissions using Nginx, Zeek, Sysmon, and Splunk. One Sysmon network event showed an unknown process, but the PID was available. I pivoted to Event ID 1 and identified curl.exe launched from PowerShell. That demonstrated how I could recover missing context by correlating multiple records."

---

# 10. Stage 7 — Reproducibility & Evidence Integrity

[View Phase 07](phase-07-reproducibility-integrity/README.md)

## Objective

Verify that the original dataset, reproduced samples, saved analysis responses, deployment identity, and recovered payload were supported by preserved evidence.

## 10.1 Image SHA-256 Verification

I executed:

```bash
sha256sum -c ~/stegashield-pilot/dataset-b-independent/evidence/SS-001-reproducibility.sha256
```

The four checked artifacts were:

- Original clean PNG
- Reproduced clean PNG
- Original modified PNG
- Reproduced modified PNG

All four returned `OK`.

**Result: 4/4 PASS**

## 10.2 Analysis JSON Verification

I also checked three saved analysis artifacts:

```text
SS-001-CLEAN-analysis.json
SS-001-LSB-RAW-analysis.json
SS-001-LSB-RAW-earlier-analysis.json
```

All three matched their recorded SHA-256 fingerprints.

**Result: 3/3 PASS**

These checks support integrity relative to the recorded manifests.

They do not independently establish original creation times or a complete forensic chain of custody.

## 10.3 Script Review

The preserved Python scripts included:

```text
generate_clean.py
embed_lsb.py
reproduce_clean.py
reproduce_lsb.py
```

The reproduction scripts wrote their output to separate paths.

Exclusive file creation helped protect existing outputs from accidental overwriting.

The reviewed methodology preserved:

- 512 × 512 RGB image dimensions
- Fixed clean-image generation seed
- 23-byte embedded payload
- RGB-LSB embedding
- 32-bit payload-length header

## 10.4 Docker Image Identity

I inspected the locally installed image and running container.

The recorded image reference was:

```text
5iprojects/stegashield:latest
```

The exact inspected image ID was:

```text
sha256:64a6a378550f479c572c689bdb0af284cd81971e591a72077a6e74f7a97ea0e5
```

The reported architecture was:

```text
arm64
```

The running container referenced the same image ID.

This improved deployment traceability compared with relying only on the mutable `latest` tag.

## 10.5 Independent Payload Recovery

I used a read-only Python extraction method to recover the RGB-LSB payload from the preserved modified image.

The procedure:

1. Read the PNG's RGB channel bytes.
2. Extract the least significant bits.
3. Decode the first 32 bits as the payload length.
4. Extract the corresponding payload bits.
5. Reconstruct the message bytes.

The recorded output was:

```text
Payload length: 23
Recovered payload: STEGASHIELD_PILOT_SS001
```

The recovered payload matched the original message.

### Interpretation

This provided independent confirmation that the modified image contained the expected hidden information.

It strengthened the false-negative finding.

The extraction relied on knowledge of the embedding format.

## 10.6 Temporary Artifact Cleanup

Temporary image copies and a certificate copy had been used during HTTPS testing.

The initial Ubuntu cleanup attempt left some files behind.

I inspected the directory, removed the remaining temporary copies, and verified that the transfer directory no longer existed.

Temporary Windows image copies and the imported test certificate trust entry were also removed.

The original dataset was preserved.

## Stage 7 Findings

| Finding | Observation | Result |
|---|---|---|
| S7-F01 | Four image checksum checks | PASS |
| S7-F02 | Three analysis checksum checks | PASS |
| S7-F03 | Running Docker image identity | Verified |
| S7-F04 | Embedded payload recovery | PASS |
| S7-F05 | Temporary transfer cleanup | Verified |

### How I Explain This to Franco

"I wanted to ensure the results were supported by preserved evidence. Four image checks and three saved analysis checks passed. I also inspected the deployed Docker image and recovered the original hidden message directly from the modified PNG."

---

# 11. Consolidated Investigation Timeline

This table describes the logical order of the laboratory work.

It is not a complete timestamped forensic chain of custody.

| Stage | Activity | Outcome |
|---|---|---|
| 1 | Laboratory monitoring verification | Telemetry verified |
| 2 | Independent image generation | Ground truth established |
| 3 | Detector deployment | Healthy API and clean baseline |
| 4 | Modified image validation | False negative observed |
| 5 | Prediction comparison | Minimal score difference |
| 6 | HTTPS and SOC investigation | Process attribution reconstructed |
| 7 | Evidence verification | Hashes passed; payload recovered |

## Recorded StegaShield Analysis Timestamps

| Analysis | Timestamp (UTC) |
|---|---|
| Clean baseline | 2026-11-17T19:42:37.944816672+00:00 |
| Earlier modified analysis | 2026-11-17T20:01:39.687291170+00:00 |
| Later modified analysis | 2026-11-17T20:05:19.979548339+00:00 |

The subsequent HTTPS telemetry investigation was a separate controlled workflow.

These baseline timestamps should not be confused with the later Nginx access-log timestamps.

---

# 12. Final Evidence Matrix

| ID | Evidence | Purpose |
|---|---|---|
| E01 | Sysmon and Splunk baseline | Monitoring readiness |
| E02 | Ground-truth manifest | Independent sample labels |
| E03 | Clean-image generator | Clean sample provenance |
| E04 | RGB-LSB embedding script | Modified sample provenance |
| E05 | Original image SHA-256 hashes | Artifact identification |
| E06 | Docker readiness evidence | Application availability |
| E07 | Clean-image analysis JSON | True-negative result |
| E08 | Modified-image analysis JSON | False-negative result |
| E09 | Earlier modified analysis JSON | Matching recorded prediction |
| E10 | Nginx access records | HTTPS application activity |
| E11 | Zeek connection records | Network metadata |
| E12 | Sysmon Event ID 3 | Endpoint network activity |
| E13 | Sysmon Event ID 1 | Process attribution |
| E14 | Reproduction scripts | Reproducibility procedure |
| E15 | Four image checksum results | Artifact verification |
| E16 | Three analysis checksum results | Response integrity |
| E17 | Docker image metadata | Deployment identity |
| E18 | Payload recovery output | Independent ground truth |
| E19 | Temporary cleanup verification | Cleanup evidence |

These entries refer to the investigation evidence.

They do not imply that every underlying raw artifact has been publicly uploaded.

The phase READMEs contain selected supporting screenshots and detailed technical findings.

---

# 13. Final Investigation Findings

## F01 — Controlled RGB-LSB False Negative

**Severity:** Not assigned; research finding rather than a confirmed production vulnerability.

**Ground truth:** MODIFIED

**Prediction:** clean

**Probability:** 0.20640557

**Supporting evidence:** Ground truth, embedding script, preserved image, saved analysis, payload recovery.

**Disposition:** Confirmed for the tested sample.

**Generalization:** Unknown.

## F02 — Limited Score Separation

**Observation:** Clean and modified probabilities differed by only 0.00000003.

**Disposition:** Verified observation.

**Root cause:** Not established.

## F03 — Incomplete Process Attribution

**Observation:** Sysmon Event ID 3 contained an unknown process image.

**Supporting fields:** Source port 54306 and PID 10344.

**Correlation:** Sysmon Event ID 1 identified `curl.exe` launched from PowerShell.

**Disposition:** Process attribution supported.

## F04 — Evidence Integrity

**Observation:** Four image checks and three saved analysis checks passed.

**Disposition:** Verified against recorded checksums.

## F05 — Payload Recovery

**Observation:** The expected 23-byte payload was recovered from the modified PNG.

**Disposition:** Verified.

---

# 14. Remaining Questions & Research Gaps

Future independent research could investigate:

1. Detection behavior with larger RGB-LSB payloads
2. Different payload densities
3. Natural photographic carrier images
4. Additional steganography algorithms
5. Repeated testing with input-byte hashes recorded
6. Effects of application preprocessing
7. Structured image-hash and analysis-ID correlation
8. Centralized Zeek and Nginx log ingestion

These are proposed research activities.

They were not completed during this pilot.

---

# 15. Recommendations for Franco's Technical Review

## Detector Evaluation

Would a larger independently generated dataset help characterize performance across different LSB densities and carrier-image types?

## Model Assumptions

Are there documented expectations regarding supported image formats, embedding methods, or preprocessing?

## Reproducibility

Could the API record each submitted image's SHA-256 fingerprint alongside its analysis identifier?

## SOC Investigation

Which structured application logs would be most valuable for correlation with endpoint and network telemetry?

## Future Testing

Which additional controlled test cases would provide the greatest technical value before making broader performance claims?

---

# 16. Five-Minute Presentation Walkthrough

This section is my speaking guide for Franco.

## Opening

"Hi Franco, thanks for meeting with me.

I completed the StegaShield Detection Validation Pilot and organized the practical work into seven stages.

My objective was to independently evaluate the detector, investigate unexpected results, correlate the associated activity using SOC telemetry, and verify the supporting evidence.

I documented the full process in GitHub so another analyst can understand the methodology and findings."

## Stage 1

"I started by verifying my SOC monitoring environment.

Sysmon network events weren't appearing as expected, so I investigated the configuration and forwarding until the required activity became visible in Splunk."

## Stage 2

"I generated my own dataset using Python and Pillow.

I created a clean RGB image and embedded a controlled 23-byte message into a modified copy.

That gave me independent ground truth."

## Stage 3

"I deployed StegaShield using Docker on Ubuntu ARM64.

The API was healthy, and the clean image was classified correctly."

## Stage 4

"I submitted the modified image containing the hidden payload.

StegaShield classified it as clean.

That was the main false-negative finding."

## Stage 5

"I compared the probabilities and found that the difference was only 0.00000003.

I documented the observation without claiming to know the internal cause."

## Stage 6

"I investigated the HTTPS submissions using Nginx, Zeek, Sysmon, and Splunk.

One network event had an unknown process image.

I pivoted to the matching process creation event and identified curl.exe launched from PowerShell."

## Stage 7

"I verified the preserved evidence.

Four image checks and three analysis JSON checks passed.

I also recovered the original 23-byte payload directly from the modified PNG."

## Closing

"My conclusion is deliberately limited.

StegaShield missed this particular independently verified RGB-LSB modification.

The investigation also demonstrated process attribution through SOC telemetry correlation and successful evidence verification.

Because the dataset was small, I cannot generalize the result to the detector's overall performance.

I'd appreciate your feedback on the methodology and what test cases you would prioritize next."

---

# 17. Questions Franco May Ask

## Q1. How do you know the modified image contained hidden data?

I generated the modified image using a documented RGB-LSB script and subsequently recovered the exact 23-byte message from the preserved PNG.

The ground truth did not rely on StegaShield's classification.

## Q2. Why is this a false negative?

The independently established ground truth was MODIFIED, but StegaShield predicted clean.

## Q3. Does this mean StegaShield is ineffective?

No.

The experiment established one missed detection under a specific controlled condition.

A larger representative dataset would be needed to assess general performance.

## Q4. Why were the probabilities almost identical?

That remains unresolved.

The observed scores were similar, but the investigation did not establish the internal cause.

## Q5. What did the SOC investigation demonstrate?

It demonstrated correlation of authorized activity across endpoint, network, and application records.

It also showed how Sysmon Event ID 1 helped resolve incomplete process attribution in Event ID 3.

## Q6. Did Zeek detect the embedded payload?

No.

Zeek recorded encrypted network connection metadata.

The payload was recovered separately from the preserved PNG.

## Q7. Was the activity malicious?

No.

All submissions were authorized laboratory tests.

The controlled steganographic payload did not establish malicious intent or exfiltration.

## Q8. What would you improve?

I would expand the independent dataset, test different carrier types and embedding conditions, record input SHA-256 hashes alongside API responses, and improve telemetry correlation.

---

# 18. Final Analyst Handoff

## Confirmed Findings

- StegaShield successfully deployed on Ubuntu ARM64
- API readiness verified
- Clean reference correctly classified
- Controlled RGB-LSB sample incorrectly classified as clean
- Original 23-byte payload independently recovered
- Nginx recorded authorized analysis requests
- Zeek recorded corresponding SSL connection metadata
- Sysmon events were searchable in Splunk
- Process attribution supported through Event IDs 1 and 3
- Four image checksum checks passed
- Three analysis JSON checksum checks passed
- Running Docker image identity verified
- Temporary transfer cleanup verified

## Not Established

- Overall StegaShield accuracy or reliability
- Detection performance across other embedding methods
- Generalization to natural photographs
- Internal root cause of the false negative
- Native StegaShield-to-Splunk integration
- Decryption of HTTPS image contents through Zeek
- Complete cross-system clock synchronization
- Complete independent forensic chain of custody

## Final Disposition

**Investigation status:** Completed.

**Primary finding:** One confirmed false negative for a controlled RGB-LSB-modified sample.

**Evidence confidence:** Supported by independent sample generation, recorded model responses, SHA-256 verification, and payload recovery.

**SOC conclusion:** Authorized image-submission activity was reconstructed through complementary endpoint, network, and application evidence.

**Recommended next action:** Review the findings with Franco and identify the next controlled test cases before drawing broader conclusions.

---

**End of Investigation Notes**

[Back to Main Project](README.md)
