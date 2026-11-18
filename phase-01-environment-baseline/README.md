
# Phase 01 — Environment & Laboratory Baseline

**StegaShield Detection Validation Pilot**

**Focus:** Laboratory readiness, endpoint telemetry, network monitoring, and Sysmon Event ID 3 investigation.

[Back to Main Project](../README.md)

---

## Objective

Before testing StegaShield, I needed to establish a reliable laboratory environment and verify that my monitoring tools were producing usable security telemetry.

My objective was not simply to confirm that the tools were installed.

I wanted to verify that network activity could be observed, Windows events could be collected, and the resulting evidence could be investigated in Splunk Enterprise.

The initial investigation focused on Windows Sysmon, Splunk, Zeek, and Docker readiness.

## Laboratory Environment

The pilot used a Mac M2 host with Windows 11 and Ubuntu virtual machines.

| Component | Technology | Purpose |
|---|---|---|
| Host | Mac M2 | Laboratory management |
| SIEM | Splunk Enterprise | Centralized event investigation |
| Windows endpoint | Windows 11 ARM64 | Controlled endpoint activity |
| Endpoint monitoring | Sysmon | Process and network telemetry |
| Log forwarding | Splunk Universal Forwarder | Windows event ingestion |
| Linux VM | Ubuntu 24.04 ARM64 | Network monitoring and analysis |
| Network sensor | Zeek | Connection and HTTP metadata |
| Container runtime | Docker | StegaShield deployment preparation |

The main telemetry path was:

Windows Activity → Sysmon → Splunk Universal Forwarder → Splunk Enterprise

Zeek independently recorded network activity visible on its monitored Ubuntu interface.

## 1. Splunk Enterprise Connectivity

I started by checking whether Splunk Enterprise was operational on my Mac.

I inspected the Splunk management listener and confirmed that the service was listening.

Next, I tested connectivity from Ubuntu to the Splunk management endpoint.

The controlled HTTPS request returned HTTP 200 OK, and the response identified Splunkd as the responding service.

This confirmed that Ubuntu could reach the Splunk management service.

![Ubuntu to Splunk Connectivity](evidence/01-splunk-connectivity.png)

**Observation:** Splunk was accessible from Ubuntu over the tested network path.

**Limitation:** Successful connectivity did not establish that Ubuntu logs were being forwarded to Splunk.

## 2. Zeek Network Monitoring

My next step was to verify that Zeek was collecting network telemetry.

I inspected the standalone Zeek configuration, checked the monitored interface, and verified the sensor's running state.

![Zeek Running Status](evidence/02-zeek-status.png)

I then inspected the connection logs.

Zeek had recorded actual network activity, including TCP, UDP, DHCP, and ICMP traffic.

![Zeek Connection Records](evidence/03-zeek-connections.png)

I also inspected the HTTP log and observed a GET request with an HTTP 204 No Content response.

These checks demonstrated that Zeek was generating network metadata.

**Observation:** Zeek was operational and producing connection and HTTP records.

**Limitation:** This did not establish complete visibility into all virtual-machine traffic or encrypted HTTPS image contents.

## 3. Sysmon Network Telemetry Investigation

This was the most important investigation in Phase 01.

I verified that Sysmon was installed and that its driver was running.

Process creation and termination events were already present.

However, I noticed that Sysmon Event ID 3, which records network connections, was missing from the inspected local event log.

This was a problem because later SOC investigations would require network connections to be associated with their originating Windows processes.

### Initial Investigation

I inspected Sysmon's active configuration.

Network connection monitoring was reported as enabled, but I still could not find the expected Event ID 3 records.

![Initial Sysmon Configuration](evidence/04-sysmon-initial-config.png)

I investigated several possible explanations, including:

- Sysmon service and driver status
- Registered driver configuration
- Driver binary and version
- Digital signature validity
- Event schema
- Retained error events
- Existing monitoring settings

The driver was running and Windows reported a valid digital signature.

A search for retained Sysmon Event ID 255 records returned no matches.

These checks did not conclusively identify the root cause.

### Remediation

I prepared a minimal Sysmon XML configuration containing explicit collection rules for:

| Event ID | Description |
|---|---|
| 1 | Process Creation |
| 3 | Network Connection |
| 5 | Process Termination |

I preserved a report of the original configuration before applying the change.

After applying the XML file, Sysmon reported that the configuration had been successfully validated and updated.

![Sysmon Configuration Update](evidence/05-sysmon-config-update.png)

### Verification

I generated another controlled TCP connection and inspected the local Sysmon Operational log.

This time, Event ID 3 records appeared.

![Local Sysmon Event ID 3](evidence/06-sysmon-event3-local.png)

The records contained information such as:

- Process ID and process GUID
- Executable path
- Source IP address and port
- Destination IP address and port
- Protocol
- Timestamp
- User context

**Finding:** Sysmon network connection telemetry was successfully restored following the configuration change.

The exact root cause of the original missing events remained inconclusive.

## 4. Splunk Forwarding Investigation

During baseline preparation, I also encountered a separate issue.

The Splunk Universal Forwarder initially received an access-denied error when attempting to subscribe to the Sysmon Operational event channel.

I investigated the forwarder service identity and event-channel permissions.

The service identity was added to the local Event Log Readers group, and the forwarder service was restarted.

Subsequent searches confirmed that Sysmon events were reaching Splunk.

I then searched for Event ID 3 records.

![Sysmon Event ID 3 in Splunk](evidence/07-splunk-event3.png)

The results showed network connection events from the Windows endpoint.

One record associated the Splunk Universal Forwarder process with an outbound connection to the Splunk receiving service.

This verified the telemetry path:

Windows Network Activity → Sysmon Event ID 3 → Splunk Universal Forwarder → Splunk Enterprise

## 5. Final Telemetry Verification

After restoring the missing telemetry, I performed a final Splunk search covering a 30-minute window.

The search returned:

| Event ID | Description | Events Observed |
|---|---|---:|
| 1 | Process Creation | 215 |
| 3 | Network Connection | 52 |
| 5 | Process Termination | 217 |
| 16 | Configuration Change | 1 |
| **Total** | | **485** |

![Final Splunk Event Coverage](evidence/08-splunk-final-events.png)

These counts represent the observed search window, not permanent system totals.

The results confirmed that process creation, network connections, process termination, and configuration changes were searchable in Splunk.

## 6. Investigation Findings

### Finding 01 — Missing Sysmon Event ID 3

**Initial observation:** Network connection events were absent from the inspected local log.

**Investigation:** Reviewed configuration, driver state, signatures, error records, and controlled network activity.

**Action:** Applied a validated Sysmon configuration with explicit collection rules.

**Verification:** Event ID 3 appeared locally and in Splunk.

**Disposition:** Resolved.

**Root cause:** Not conclusively established.

### Finding 02 — Splunk Forwarding Access Denied

**Initial observation:** The forwarder could not subscribe to the Sysmon Operational channel.

**Investigation:** Reviewed service identity and event-channel permissions.

**Action:** Updated Event Log Readers membership and restarted the forwarder.

**Verification:** Sysmon records became searchable in Splunk.

**Disposition:** Resolved.

### Finding 03 — Network Visibility Boundaries

**Observation:** Zeek generated actual network connection and HTTP metadata.

**Limitation:** Complete virtual-machine traffic visibility and encrypted image-content inspection were not established.

**Disposition:** Documented for subsequent investigation phases.

## 7. What I Discovered

The most important lesson from Phase 01 was that an enabled security feature does not automatically guarantee usable telemetry.

Initially, Sysmon was running and network monitoring was reported as enabled.

However, Event ID 3 was still missing.

I had to investigate the configuration, perform controlled network activity, verify local event generation, and then confirm centralized ingestion in Splunk.

This taught me to distinguish between three questions:

1. Did the activity happen?
2. Did the endpoint generate the expected event?
3. Did the SIEM successfully receive and index that event?

Each question requires separate evidence.

## 8. Limitations

The investigation did not conclusively establish the original cause of the missing Sysmon Event ID 3 events.

Zeek's operational status did not prove complete visibility into every laboratory network path.

Ubuntu log forwarding to Splunk was not independently verified during this baseline.

StegaShield detection performance was outside the scope of Phase 01.

## Conclusion

Phase 01 established a working laboratory telemetry baseline.

I verified Splunk connectivity, operational Zeek logging, Windows Sysmon events, and centralized Sysmon ingestion.

The investigation identified and resolved two separate problems affecting network event collection and forwarding.

The main lesson was that security monitoring must be verified through actual events rather than assumptions about tool configuration.

**Phase 01 Outcome: Core technical baseline verified.**

**Evidence publication status:** The original Phase 01 screenshots remain pending recovery and upload. The image references above identify the intended evidence locations and will render only after genuine screenshots are added.

---

**Next: [Phase 02 — Independent Dataset & Ground Truth](../phase-02-independent-dataset/README.md)**

[Back to Main Project](../README.md)


