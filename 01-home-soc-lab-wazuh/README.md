# Project 1: Home SOC Lab with Wazuh

## Objective

Build a small home Security Operations Center (SOC) lab to learn SIEM deployment, endpoint monitoring, alert review, and basic security investigations.

## Lab environment

- Host operating system: Windows 11
- Virtualization: Oracle VirtualBox
- SIEM host: Kali Linux VM running the official Wazuh single-node Docker deployment
- Endpoint: Windows 11 host with Wazuh agent 4.14.9 installed
- Network: VirtualBox NAT plus Host-only networking

## Deployment

The Wazuh 4.14.9 single-node Docker deployment was taken from the official [Wazuh Docker repository](https://github.com/wazuh/wazuh-docker), using its `single-node` configuration.

The deployment includes:
- Wazuh Manager
- Wazuh Indexer
- Wazuh Dashboard

The dashboard was opened successfully from Kali at `https://localhost`. Container status showed all three services running, and the Indexer logs reported cluster health GREEN.

## Network configuration

The Kali VM used:
- NAT interface: `10.0.2.15`
- Host-only interface: `192.168.56.101`

The Windows host successfully reached the Kali host-only address. TCP connectivity tests to ports 1514, 1515, and 55000 succeeded.

## Windows endpoint

The Wazuh Windows agent 4.14.9 MSI was installed through the Windows installer, and the Wazuh service was started.

**Enrollment and event ingestion are not yet verified.** The next step is to confirm that the endpoint appears as Active in the Wazuh dashboard, then generate and investigate a safe test event.

## Resource constraints

The Kali VM has 4 GB RAM and 4 virtual CPUs. This is below the official 8 GB RAM recommendation for the all-in-one Wazuh deployment. During startup, the VM showed swap usage. The physical host also has 8 GB RAM, so running the SIEM alongside Windows applications can cause memory pressure.

## Security and evidence handling

- The lab is isolated using VirtualBox networking.
- Do not expose the dashboard or management ports to the public internet.
- Do not commit passwords, API keys, private keys, certificates, or real endpoint data.
- Default lab credentials must be changed before any external exposure.

## Next steps

1. Verify Windows agent enrollment and active status.
2. Confirm that Windows events are reaching Wazuh.
3. Generate a controlled test event and locate its alert or log record.
4. Document the investigation with sanitized evidence.
5. Record tuning, lessons learned, and limitations.

## Status

Deployment is running; endpoint enrollment and event ingestion remain to be verified.
