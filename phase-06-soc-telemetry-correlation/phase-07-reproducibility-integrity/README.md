
# Phase 07 — Reproducibility & Evidence Integrity

**StegaShield Detection Validation Pilot**

**Focus:** Dataset reproducibility, SHA-256 verification, deployment identity, payload recovery, evidence preservation, and temporary-file cleanup.

[Back to Main Project](../README.md)

---

## 1. Objective

After completing the StegaShield detection tests and SOC telemetry investigation, I wanted to verify the integrity of the retained research artifacts.

My objective was to ensure that the original image samples were preserved, the generation methodology could be reproduced, and the saved analysis responses could be checked against their recorded SHA-256 fingerprints.

I also wanted to document the exact Docker image used during the experiment, independently recover the embedded payload, and remove temporary transfer files without disturbing the original dataset.

The investigation focused on six questions:

1. Were the original clean and modified images still intact?
2. Did the reproduced images match their recorded SHA-256 fingerprints?
3. Were the saved StegaShield analysis records unchanged relative to their recorded checksums?
4. Could I identify the exact Docker image used by the running container?
5. Could I recover the original payload from the preserved modified image?
6. Could temporary transfer artifacts be removed while preserving the research evidence?

This phase was an integrity and reproducibility review, not an additional detection benchmark.

## 2. Laboratory Environment

The verification was conducted on the existing Ubuntu ARM64 laboratory server.

| Component | Technology | Purpose |
|---|---|---|
| Laboratory server | Ubuntu ARM64 | Evidence verification |
| Detection service | StegaShield | Previously tested image-analysis application |
| Container runtime | Docker | Image and container identity inspection |
| Image processing | Python and Pillow | Reproduction and payload extraction |
| Integrity verification | SHA-256 | Artifact fingerprint comparison |
| Dataset | Independent Dataset B | Original and reproduced samples |
| Evidence format | PNG, JSON, CSV, Python | Preserved research artifacts |

The original dataset was located under:

```text
/home/james/stegashield-pilot/dataset-b-independent/
```

I performed verification against the preserved files rather than generating replacement evidence.

## 3. Evidence Preservation Approach

The independent dataset had already been established during Phase 02.

It contained two primary samples:

| Sample ID | Ground Truth | Purpose |
|---|---|---|
| SS-001-CLEAN | CLEAN | Known-clean reference |
| SS-001-LSB-RAW | MODIFIED | Controlled RGB-LSB steganography sample |

The modified sample originated from the clean reference and contained a 23-byte raw payload.

To protect the original samples, I used separate reproduction output paths.

The Python scripts also used exclusive file creation:

```python
with OUTPUT.open("xb") as file:
    image.save(file, format="PNG")
```

This approach prevents the script from silently overwriting an existing file at the specified output path.

It does not make the entire dataset immutable, but it provides protection against accidental overwrites during normal script execution.

## 4. Temporary Transfer Cleanup

During the earlier HTTPS testing, I temporarily copied two image files and a test certificate into a dedicated Ubuntu transfer directory.

The temporary directory was:

```text
/home/james/stegashield-cert-transfer/
```

Its contents included:

- SS-001-CLEAN.png
- SS-001-LSB-RAW.png
- stegashield-server.crt

These were transfer copies, not the original Dataset B files.

### Initial Cleanup Attempt

I initially used interactive removal to delete the temporary directory contents.

The first cleanup attempt did not remove every file.

A subsequent directory inspection confirmed that the clean-image copy and certificate copy remained.

I continued by removing those remaining temporary files individually, then inspected the directory again.

Once it was empty, I removed the directory using `rmdir`.

### Final Verification

I verified that the temporary directory no longer existed.

The terminal returned:

```text
PASS: Temporary transfer directory removed
```

![Temporary Transfer Cleanup Verified](evidence/01-transfer-cleanup-verified.png)

### Finding P07-F01 — Temporary Transfer Cleanup

**Initial condition:** Temporary transfer files remained after the first cleanup attempt.

**Action:** Removed the remaining copies and the empty transfer directory.

**Verification:** The directory-absence test passed.

**Disposition:** Resolved.

### Important Distinction

The cleanup affected only the temporary transfer directory.

The original Dataset B images and reproducibility evidence were not intentionally removed.

The running Nginx TLS certificate under `/etc/nginx/stegashield-tls/` was separate from the temporary certificate copy.

## 5. Dataset Image Integrity Verification

After cleanup, I verified the recorded SHA-256 fingerprints of the original and reproduced image artifacts.

The existing checksum file was:

```text
dataset-b-independent/evidence/SS-001-reproducibility.sha256
```

I executed:

```bash
sha256sum -c ~/stegashield-pilot/dataset-b-independent/evidence/SS-001-reproducibility.sha256
```

### Recorded Results

```text
SS-001-CLEAN.png: OK
SS-001-REPRODUCED.png: OK
SS-001-LSB-RAW.png: OK
SS-001-LSB-REPRODUCED.png: OK
```

![Four Image SHA-256 Checks Passed](evidence/02-four-image-checksums-pass.png)

### Verification Summary

| Artifact | Checksum Result |
|---|---|
| Original clean PNG | PASS |
| Reproduced clean PNG | PASS |
| Original RGB-LSB PNG | PASS |
| Reproduced RGB-LSB PNG | PASS |
| **Total** | **4/4 PASS** |

### Interpretation

All four image files matched the expected SHA-256 values recorded in the reproducibility checksum manifest.

This supported the integrity of the preserved artifacts relative to that manifest.

The successful checks did not independently establish when the original files or checksum records were first created.

The verification also did not constitute a complete forensic chain of custody.

## 6. Ground-Truth Manifest Review

I inspected the preserved ground-truth manifest:

```text
dataset-b-independent/evidence/ground-truth-manifest.csv
```

The file contained two sample records and twelve columns.

The labels were:

```text
SS-001-CLEAN → CLEAN
SS-001-LSB-RAW → MODIFIED
```

### Clean Reference

The manifest recorded:

| Property | Value |
|---|---|
| Sample ID | SS-001-CLEAN |
| Parent ID | NONE |
| Format | PNG |
| Mode | RGB |
| Width | 512 |
| Height | 512 |
| Embedding method | NONE |
| Payload bytes | 0 |

The recorded SHA-256 fingerprint was:

```text
d6fc09cec7ccd77d509457f610ab1d3f73a9e68dc3bbc146cb383e5255f34d5a
```

### Modified Sample

The manifest recorded:

| Property | Value |
|---|---|
| Sample ID | SS-001-LSB-RAW |
| Parent ID | SS-001-CLEAN |
| Format | PNG |
| Mode | RGB |
| Width | 512 |
| Height | 512 |
| Embedding method | RGB-LSB |
| Payload preparation | RAW |
| Payload bytes | 23 |

The recorded SHA-256 fingerprint was:

```text
a13757e106766d7d4960c151158c56131245646ef4cb50f28208649d4db97570
```

### Observation

The preserved manifest maintained the relationship between the clean image and its modified derivative.

The two original samples had different SHA-256 fingerprints.

The manifest was previously validated during dataset preparation.

Its labels originated from the controlled image-generation methodology, not from StegaShield's predictions.

## 7. Saved Analysis JSON Integrity

I next reviewed the saved StegaShield analysis artifacts.

The evidence directory contained three retained analysis responses:

```text
SS-001-CLEAN-analysis.json
SS-001-LSB-RAW-analysis.json
SS-001-LSB-RAW-earlier-analysis.json
```

Each file had a corresponding SHA-256 checksum record.

I executed the integrity verification from the evidence directory.

```bash
cd ~/stegashield-pilot/dataset-b-independent/evidence && sha256sum -c SS-001-CLEAN-analysis.sha256 SS-001-LSB-RAW-analysis.sha256 SS-001-LSB-RAW-earlier-analysis.sha256
```

### Recorded Results

All three files returned `OK`.

![Saved Analysis JSON Integrity Verification](evidence/03-three-analysis-json-checksums-pass.png)

| Saved Analysis | Result |
|---|---|
| Clean-image JSON | PASS |
| Modified-image JSON | PASS |
| Earlier modified-image JSON | PASS |
| **Total** | **3/3 PASS** |

### Preserved Classification Results

The saved records contained:

| Analysis | Ground Truth | Prediction | Probability |
|---|---|---|---|
| Clean image | CLEAN | clean | 0.20640554 |
| Modified image | MODIFIED | clean | 0.20640557 |
| Earlier modified-image analysis | MODIFIED sample filename | clean | 0.20640557 |

The clean-image prediction was a true negative.

The modified-image prediction was a false negative for the controlled sample.

The two modified-image records had different analysis IDs and timestamps but identical reported predictions and probabilities.

### Finding P07-F02 — Analysis Artifact Integrity

**Observation:** Three saved JSON responses matched their recorded SHA-256 checksums.

**Interpretation:** The retained analysis records passed the performed integrity checks.

**Limitation:** The JSON responses did not include independent input-image SHA-256 fields that would establish exact input-byte identity across every API submission.

## 8. Docker Deployment Identity

The original deployment configuration referenced:

```text
5iprojects/stegashield:latest
```

The `latest` tag identifies a mutable image reference.

For reproducibility, I needed to identify the exact image available in the laboratory rather than rely on the tag alone.

### Local Image Inspection

I inspected the Docker image metadata using:

```bash
sudo docker image inspect 5iprojects/stegashield:latest --format 'Image ID: {{.Id}}{{println}}Repo Digests: {{json .RepoDigests}}{{println}}Architecture: {{.Architecture}}'
```

The output identified the image as:

```text
sha256:64a6a378550f479c572c689bdb0af284cd81971e591a72077a6e74f7a97ea0e5
```

The reported architecture was:

```text
arm64
```

The repository digest was recorded with the same SHA-256 value.

![StegaShield Docker Image Identity](evidence/04-docker-image-identity.png)

### Running Container Inspection

I separately inspected the deployed container:

```bash
sudo docker inspect stegashield --format 'Container: {{.Name}}{{println}}Running image ID: {{.Image}}{{println}}State: {{.State.Status}}'
```

The output showed:

```text
Container: /stegashield
State: running
```

The running container's image ID matched the previously inspected local image ID.

![Running Container Image Verification](evidence/05-running-container-image-match.png)

### Finding P07-F03 — Deployed Image Identified

**Observation:** The local image and running container referenced the same image ID.

**Architecture:** ARM64.

**State at verification:** Running.

**Disposition:** Verified.

### Reproducibility Significance

Recording the exact image ID improved deployment traceability.

It reduced reliance on the mutable `latest` tag when describing the tested installation.

However, this inspection did not independently reveal or audit the StegaShield model's internal weights, training dataset, or architecture.

## 9. Clean Image Reproduction Script Review

I reviewed:

```text
dataset-b-independent/scripts/reproduce_clean.py
```

The script used Python and Pillow to recreate the synthetic clean image.

### Verified Parameters

```python
WIDTH = 512
HEIGHT = 512
SEED = 20261007
```

The generator used:

```python
rng = Random(SEED)
```

The script generated RGB channel values using the same deterministic procedure as the original clean-image generator.

Its output was:

```text
evidence/SS-001-REPRODUCED.png
```

![Clean Image Reproduction Script](evidence/06-clean-reproduction-script.png)

### Observation

The reproduction script preserved the original image dimensions, mode, seed, and generation approach.

It wrote the reproduced image into the evidence directory instead of overwriting the original clean sample.

The subsequent SHA-256 check returned `OK` for the reproduced image.

### Important Limitation

The checksum check establishes a match against the recorded expected digest.

The screenshot reviewed in this phase does not by itself establish reproducibility across different Python or Pillow versions.

## 10. RGB-LSB Image Reproduction Script Review

I also reviewed:

```text
dataset-b-independent/scripts/reproduce_lsb.py
```

The script loaded the preserved clean image and applied the same RGB-LSB embedding procedure used during Dataset B creation.

### Verified Parameters

| Property | Value |
|---|---|
| Source | SS-001-CLEAN.png |
| Payload | STEGASHIELD_PILOT_SS001 |
| Payload length | 23 bytes |
| Header | 32-bit length |
| Embedding method | RGB-LSB |
| Output | SS-001-LSB-REPRODUCED.png |

The script converted the payload and length header into binary values.

It then modified the least significant bits of consecutive RGB channel bytes.

The central operation was:

```python
channels[index] = (channels[index] & 254) | int(bit)
```

The output was written separately:

```text
evidence/SS-001-LSB-REPRODUCED.png
```

![RGB-LSB Reproduction Script](evidence/07-lsb-reproduction-script.png)

### Observation

The reproduction script preserved the original embedding method and payload.

It used exclusive file creation to prevent overwriting an existing output file.

The reproduced modified PNG passed its recorded SHA-256 integrity check.

### Finding P07-F04 — Reproduction Artifacts Verified

The generation and reproduction scripts documented how the two image conditions were constructed.

Both reproduced image artifacts passed their recorded integrity checks.

This supported the repeatability and traceability of the controlled test methodology.

## 11. Independent Embedded Payload Recovery

One of the most important validation checks was confirming that the preserved modified image genuinely contained the expected payload.

The model had classified the modified sample as `clean`.

I therefore performed an independent, read-only extraction of the RGB least-significant bits.

### Extraction Method

I used Python and Pillow to:

1. Open the preserved `SS-001-LSB-RAW.png` image.
2. Read its RGB channel bytes.
3. Extract the least significant bit of each channel byte.
4. Interpret the first 32 bits as the payload-length header.
5. Read the corresponding payload bits.
6. Reconstruct and decode the payload bytes.

This was a direct verification of the documented embedding format.

### Recorded Output

The terminal returned:

```text
Payload length: 23
Recovered payload: STEGASHIELD_PILOT_SS001
```

![Embedded Payload Recovery](evidence/08-embedded-payload-recovered.png)

### Finding P07-F05 — Payload Successfully Recovered

**Expected payload:** `STEGASHIELD_PILOT_SS001`

**Recovered payload:** `STEGASHIELD_PILOT_SS001`

**Recovered length:** 23 bytes.

**Disposition:** Verified.

### Significance

The successful extraction established that the preserved modified PNG contained the expected embedded message in its RGB least-significant bits.

This strengthened the ground-truth evidence supporting the false-negative detection result.

The extraction did not overwrite the modified image.

### Boundary

Successful recovery was achieved using knowledge of the embedding format.

It does not establish that an arbitrary unknown payload or embedding method could be detected or extracted without such knowledge.

## 12. Evidence Inventory

I inspected the preserved project directories and evidence files.

The evidence directory contained:

- Ground-truth manifest.
- Original clean-image checksum.
- Original modified-image checksum.
- Original and reproduced image artifacts.
- Reproducibility checksum manifest.
- Clean-image analysis JSON.
- Modified-image analysis JSON.
- Earlier modified-image analysis JSON.
- Corresponding analysis checksums.

The scripts directory contained:

```text
embed_lsb.py
generate_clean.py
reproduce_clean.py
reproduce_lsb.py
```

The deployment directory contained:

```text
compose.yaml
.env
stegashield-server.crt
```

The `.env` file was retained locally for deployment use and must not be published to GitHub.

The original private deployment credentials and TLS private key are not part of the public evidence package.

### Documentation Checkpoint

During this phase, I searched the pilot directory for existing Markdown, text, and PDF documentation files.

That search returned no matching documents.

I subsequently created a dedicated documentation directory for the project.

The project documentation is being prepared separately from the original validation evidence.

## 13. Investigation Findings

| Finding ID | Observation | Status |
|---|---|---|
| P07-F01 | Temporary transfer directory removed and absence verified | PASS |
| P07-F02 | Four image SHA-256 checks matched recorded values | 4/4 PASS |
| P07-F03 | Three analysis JSON checks matched recorded values | 3/3 PASS |
| P07-F04 | Local Docker image ID matched running container image ID | Verified |
| P07-F05 | Generation and reproduction scripts reviewed | Verified |
| P07-F06 | Controlled 23-byte payload successfully recovered | PASS |
| P07-F07 | Complete independent forensic chain of custody | Not established |

## 14. What I Discovered

### Discovery 01 — Cleanup Must Be Verified

The first removal attempt did not eliminate every temporary transfer file.

I discovered this by checking whether the directory still existed and listing its contents.

The final cleanup was only considered complete after the directory-absence test passed.

### Discovery 02 — Reproducibility Requires Preserved Inputs

Keeping original and reproduced images separate made it possible to examine and verify each artifact independently.

The scripts also protected existing output files against accidental replacement.

### Discovery 03 — Checksums Support Integrity, Not Every Aspect of Provenance

All seven checksum checks passed.

This established agreement between the preserved files and their recorded fingerprints.

However, checksums alone do not establish when the files were created or provide a complete independent chain of custody.

### Discovery 04 — The Running Image Matters

Recording `latest` is not sufficient for precise long-term deployment identification.

Inspecting the Docker image and running container provided an exact image identity for the tested environment.

### Discovery 05 — Ground Truth Can Be Verified Beyond Labels

The successful recovery of the 23-byte payload demonstrated that the preserved modified image contained the expected embedded information.

This provided stronger evidence than relying on the MODIFIED label alone.

## 15. Scope and Limitations

This phase verified the integrity and reproducibility evidence for the existing controlled experiment.

It did not evaluate additional StegaShield samples or embedding methods.

The checksum comparisons relied on previously recorded manifests.

No independent timestamped evidence-management service or complete forensic chain of custody was established.

The original and reproduced samples were generated in the controlled laboratory environment.

Cross-platform reproducibility across different software versions was not established.

The payload extraction relied on the known RGB-LSB embedding format.

The Docker inspection identified the deployed image but did not independently audit the model internals.

These boundaries are important when interpreting the verification results.

## 16. Final Outcome

Phase 07 completed the integrity and reproducibility review of the StegaShield pilot.

I verified four image files and three saved analysis JSON records against their recorded SHA-256 checksums.

I reviewed the generation and reproduction scripts, confirmed the deployed Docker image identity, and successfully recovered the original 23-byte RGB-LSB payload.

I also removed the temporary Ubuntu transfer files and verified that the transfer directory no longer existed.

The original dataset and supporting laboratory artifacts remained available for inspection.

The key lesson was that a defensible detection-validation finding requires more than a model prediction.

It requires preserved samples, documented methods, integrity verification, repeatable procedures, and transparent limitations.

**Phase 07 Status: Reproducibility & Evidence Integrity Review Completed.**

**Image Integrity: 4/4 PASS.**

**Analysis JSON Integrity: 3/3 PASS.**

**Payload Recovery: PASS.**

**Temporary Transfer Cleanup: VERIFIED.**

---

## Next Phase

[Phase 08 — Consolidated Technical Assessment](../phase-08-final-assessment/README.md)

[Back to Main Project](../README.md)
