
# Phase 01 — Environment & Laboratory Baseline

**StegaShield Detection Validation Pilot**

**Focus:** Laboratory readiness, network monitoring, Windows endpoint telemetry, and Sysmon Event ID 3 investigation.

[← Back to Main Project](../README.md)

---

## 1. Objective

Before testing StegaShield, I needed to establish a reliable SOC laboratory and verify that my security monitoring tools were collecting usable telemetry.

My goal was not simply to confirm that Splunk, Sysmon, Zeek, and Docker were installed.

I wanted to establish that:

- The laboratory systems could communicate.
- Splunk Enterprise was operational.
- Windows endpoint events were being collected.
- Sysmon network connection events were available.
- Windows telemetry was searchable in Splunk.
- Zeek was generating network metadata.
- Docker was ready for the later StegaShield deployment.

This baseline was important because the later detection experiments would depend on reliable endpoint and network evidence.

## 2. Laboratory Environment

The pilot used a Mac M2 host with Windows 11 and Ubuntu ARM64 virtual machines.

| Component | Technology | Purpose |
|---|---|---|
| Host workstation | Mac M2 | Laboratory management |
| SIEM | Splunk Enterprise | Centralized investigation |
| Windows endpoint | Windows 11 ARM64 | Controlled endpoint activity |
| Endpoint monitoring | Sysmon 15.22 | Process and network telemetry |
| Log forwarding | Splunk Universal Forwarder | Windows event ingestion |
| Linux system | Ubuntu 24.04 ARM64 | Analysis and monitoring |
| Network sensor | Zeek 9.0.1 | Network connection and HTTP metadata |
| Container runtime | Docker | StegaShield deployment preparation |

The primary endpoint monitoring path was:

Windows Activity → Sysmon → Splunk Universal Forwarder → Splunk Enterprise

Zeek independently collected network metadata visible on its configured Ubuntu interface.

## 3. Splunk Enterprise Verification

I started by verifying that Splunk Enterprise was running on my Mac.

I inspected the Splunk management listener and confirmed that the service was listening.

Next, I tested connectivity from Ubuntu to the Splunk management endpoint.

The controlled HTTPS request returned HTTP 200 OK.

The response headers also identified Splunkd as the responding service.

### Observation

Ubuntu could successfully communicate with the Splunk management service.

### Interpretation

The tested network path was operational, and the Splunk management endpoint was accessible.

However, successful connectivity alone did not establish Windows event ingestion or Ubuntu log forwarding.

These needed separate verification.

## 4. Zeek Network Monitoring Verification

My next task was to verify the availability of network telemetry.

I inspected Zeek's standalone configuration and checked the monitored Ubuntu interface.

After starting Zeek, I verified its operational status through ZeekControl.

The output confirmed that the standalone Zeek instance was running.

### Connection Logging

I inspected Zeek's connection log.

The records included:

- TCP connections
- Rejected TCP connections
- UDP activity
- DHCP traffic
- ICMP traffic

This confirmed that Zeek was generating actual connection records rather than only displaying log headers.

### HTTP Logging

I also inspected the Zeek HTTP log.

One observed transaction contained:

| Field | Observed Value |
|---|---|
| Method | GET |
| Request path | / |
| Response code | 204 |
| Response message | No Content |

The activity was consistent with routine connectivity checking.

### Discovery

Zeek was operational and producing network metadata.

However, its running state did not establish visibility into every virtual-machine network path.

It also did not demonstrate inspection of encrypted HTTPS image contents.

## 5. Sysmon Event ID 3 Investigation

This became the most important investigation in Phase 01.

While verifying Windows endpoint telemetry, I confirmed that Sysmon was installed and running.

Process creation and termination events were present in the local Sysmon Operational log.

However, network connection Event ID 3 was missing.

This was a significant visibility gap because I needed network connection telemetry for the later SOC investigations.

### 5.1 Initial Observation

The Windows endpoint could establish network connections.

Sysmon was running.

The active configuration reported network connection monitoring as enabled.

Despite these observations, the inspected log contained no matching Event ID 3 records.

This raised an important investigation question:

**Why was successful network activity not producing the expected endpoint telemetry?**

### 5.2 Investigation

Before making configuration changes, I examined several possible explanations.

I checked:

- Sysmon service status
- Sysmon driver status
- Registered driver configuration
- Executable and driver versions
- Driver binary presence
- Driver digital signature
- Active event collection configuration
- Supported Sysmon event schema
- Retained Sysmon error records

The driver was running, and Windows reported a valid digital signature.

A query for retained Event ID 255 records returned no matches.

These observations did not establish the exact root cause.

### 5.3 Configuration Remediation

I prepared a minimal Sysmon XML configuration containing explicit collection rules for three event types.

| Event ID | Description |
|---|---|
| 1 | Process Creation |
| 3 | Network Connection |
| 5 | Process Termination |

Before applying the configuration, I preserved a report of the original active settings.

I then applied the new XML configuration.

Sysmon reported:

```text
Configuration file validated.
Configuration updated.
```

I inspected the active configuration again and confirmed that the intended event collection rules were present.

### 5.4 Controlled Network Test

After applying the configuration, I generated another controlled TCP connection to an existing laboratory service.

The network connection succeeded.

I searched the local Sysmon Operational log for recent Event ID 3 records.

This time, actual network connection events appeared.

The events provided details including:

- Timestamp
- Process GUID
- Process ID
- Executable path
- User context
- Protocol
- Source address and port
- Destination address and port

### Finding

**Finding ID:** P01-F01

**Title:** Missing Sysmon Network Connection Telemetry

**Initial observation:** Event ID 3 was absent from the inspected retained log.

**Action:** Applied a validated configuration with explicit event collection rules.

**Verification:** Event ID 3 appeared locally following the configuration change.

**Disposition:** Resolved.

**Root cause:** Not conclusively established.

### What I Learned

Successful connectivity does not guarantee that the expected security event has been generated.

A SOC analyst must verify actual telemetry rather than assume that an enabled monitoring setting is functioning correctly.

## 6. Splunk Universal Forwarder Investigation

During baseline preparation, I also encountered a separate issue affecting centralized event ingestion.

The Splunk Universal Forwarder initially received an access-denied error when attempting to subscribe to the Sysmon Operational event channel.

This was different from the missing local Event ID 3 problem.

The investigation required distinguishing between event generation and log forwarding.

### Investigation and Remediation

I reviewed the forwarder service identity and event-channel access permissions.

The forwarder service identity was added to the local Event Log Readers group.

I then restarted the forwarder service.

Subsequent Splunk searches confirmed that Sysmon telemetry was being ingested.

### Finding

**Finding ID:** P01-F02

**Title:** Sysmon Event Channel Access Denied

**Initial observation:** The forwarder could not subscribe to the Sysmon Operational channel.

**Action:** Updated event log access permissions and restarted the service.

**Verification:** Sysmon events became searchable in Splunk.

**Disposition:** Resolved.

### SOC Lesson

There are separate stages in the telemetry pipeline:

1. Security activity occurs.
2. The endpoint generates an event.
3. The log forwarder accesses the event.
4. The forwarder sends the event.
5. The SIEM indexes the event.
6. The analyst searches and interprets the event.

A failure at any stage can create a monitoring blind spot.

## 7. Sysmon Event ID 3 Verification in Splunk

After restoring local network telemetry, I investigated whether those records were available centrally.

I searched Splunk for Sysmon Event ID 3.

The search returned Windows network connection events.

The available records included:

- Host
- Process image
- Source address
- Destination address
- Destination port
- Protocol
- Event timestamp

One observed connection was associated with the Splunk Universal Forwarder communicating with the Splunk receiving service.

Other network activity included browser-related connections.

### Result

I successfully verified the endpoint telemetry path:

Windows Network Activity → Sysmon Event ID 3 → Splunk Universal Forwarder → Splunk Enterprise

This confirmed that Event ID 3 was being generated locally and was searchable in Splunk.

## 8. Final Telemetry Verification

I performed a final Splunk search covering a 30-minute window.

The investigation returned the following event counts:

| Event ID | Description | Count |
|---|---|---:|
| 1 | Process Creation | 215 |
| 3 | Network Connection | 52 |
| 5 | Process Termination | 217 |
| 16 | Configuration Change | 1 |
| **Total** | | **485** |

The results confirmed that process creation, network connections, process termination, and configuration changes were available in Splunk.

These counts represent the observed 30-minute search window.

They are not permanent event totals or detection accuracy measurements.

## 9. Docker Readiness

I also checked the Docker installation on Ubuntu in preparation for the later StegaShield deployment.

The Docker service was active.

The container inventory showed no deployed containers at the time of the baseline inspection.

This confirmed basic container runtime readiness.

StegaShield deployment, API functionality, and detection accuracy were not evaluated during this phase.

## 10. Investigation Summary

| Finding | Observation | Disposition |
|---|---|---|
| P01-F01 | Missing Sysmon Event ID 3 | Resolved |
| P01-F02 | Splunk forwarding access denied | Resolved |
| P01-F03 | Limited network visibility guarantees | Documented limitation |

### Verified

- Splunk Enterprise service accessibility
- Ubuntu connectivity to the Splunk management endpoint
- Sysmon service and driver operation
- Sysmon process telemetry
- Local Event ID 3 generation after remediation
- Event ID 3 ingestion into Splunk
- Splunk Event IDs 1, 3, 5, and 16
- Zeek standalone operation
- Zeek connection logging
- Zeek HTTP logging
- Docker runtime readiness

### Not Established

- Definitive root cause of the original missing Event ID 3
- Complete visibility into every virtual-machine network path
- Ubuntu Zeek log forwarding into Splunk
- StegaShield API functionality
- Steganography detection performance
- HTTPS image-content inspection

## 11. Key Discoveries

### Discovery 01 — Tool Status Is Not Telemetry Verification

Sysmon was running, but the expected network connection events were initially absent.

This demonstrated that service availability and telemetry availability are different.

### Discovery 02 — Local Events and SIEM Ingestion Are Separate

I encountered both an event collection problem and an event-channel access problem.

Resolving one did not automatically resolve the other.

### Discovery 03 — Correlation Depends on Data Quality

Before investigating suspicious network activity, I needed confidence that endpoint and network records were actually available.

Without reliable telemetry, an investigation can produce incomplete or misleading conclusions.

## 12. Final Outcome

Phase 01 established the core laboratory monitoring baseline for the StegaShield pilot.

I verified that Splunk was accessible, Zeek was generating network records, and Windows Sysmon telemetry was being forwarded successfully.

The investigation identified and resolved two separate telemetry problems.

Most importantly, I restored Sysmon Event ID 3 visibility and confirmed that network connection records were available locally and in Splunk.

The exact cause of the original missing network events remained unresolved.

### Final Lesson

**An enabled security control does not guarantee useful evidence.**

Before trusting a detection or beginning an investigation, verify the complete telemetry path using actual controlled activity.

**Phase 01 Status: Core Technical Baseline Verified.**

---

## Next Phase

[Phase 02 — Independent Dataset & Ground Truth](../phase-02-independent-dataset/README.md)

[← Back to Main Project](../README.md)
