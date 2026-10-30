# Wazuh Lab: Network and Setup Notes

## VirtualBox networking

The Kali VM uses two adapters:

| Adapter | Mode | Purpose |
|---|---|---|
| 1 | NAT | Internet access for package and image downloads |
| 2 | Host-only | Private communication between Windows host and Kali VM |

Observed Kali addresses:

| Interface | Address | Role |
|---|---|---|
| eth0 | 10.0.2.15/24 | NAT |
| eth1 | 192.168.56.101/24 | Host-only |

The Windows host successfully pinged `192.168.56.101`.

## Wazuh ports tested from Windows

The following PowerShell checks returned `TcpTestSucceeded : True`:

```powershell
Test-NetConnection 192.168.56.101 -Port 1514
Test-NetConnection 192.168.56.101 -Port 1515
Test-NetConnection 192.168.56.101 -Port 55000
```

These ports are used by Wazuh agent communication, enrollment, and the Wazuh API respectively.

## Deployment commands

The official Wazuh Docker repository was cloned at tag `v4.14.9`. Commands used from its `single-node` directory:

```bash
sudo docker-compose config --quiet
sudo docker-compose -f generate-indexer-certs.yml run --rm generator
sudo docker-compose up -d
sudo docker ps -a
```

This environment uses the standalone `docker-compose` command.

The host kernel setting required by the Indexer was applied with:

```bash
sudo sysctl -w vm.max_map_count=262144
```

This runtime setting may need to be reapplied after a reboot unless made persistent.

## Verification notes

- Compose configuration validation completed without errors.
- Certificate generation completed and the expected certificate files were created.
- The Manager, Indexer, and Dashboard containers were observed running.
- Indexer logs reported cluster health GREEN.
- The Wazuh dashboard loaded locally over HTTPS.
- Windows-to-Kali connectivity checks for ports 1514, 1515, and 55000 succeeded.
- The Windows Wazuh agent service was started.
- Agent enrollment and event ingestion have not yet been confirmed.

## Resource observations

The Kali VM was configured with 4 GB RAM and 4 vCPUs. At one observation after startup, approximately 1.3 GB was reported as available and around 463 MB of swap was in use. These figures are point-in-time observations, not performance benchmarks.

The Wazuh all-in-one deployment's official recommended RAM is higher than this VM allocation. Host memory pressure should be monitored, especially when Chrome and other applications are open in Windows.
