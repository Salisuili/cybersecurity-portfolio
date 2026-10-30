# Project 1 — Wazuh Home SOC Lab: Step-by-Step Build Guide

This guide records the commands used in this lab, what each command does, why the step matters, and how to verify the result. It distinguishes completed steps from checks that are still pending.

> **Actual deployment:** Wazuh is running on Kali Linux through Docker Compose. Windows 11 is the monitored endpoint. Ubuntu Server was installed as a separate VM but was not used to deploy or run Wazuh; it remains a powered-off option for a possible future migration.

> **Lab scope:** Use only your own Windows host and VirtualBox VMs. Keep the lab on NAT/Host-only networking. Do not expose Wazuh ports or the dashboard to the public internet.

## 1. Prepare the virtual machines

### 1.1 Kali Linux VM

Kali is the Wazuh all-in-one host in this build.

Observed VM allocation:
- RAM: 4 GB
- CPUs: 4
- Virtual disk: 80 GB
- Network adapter 1: NAT
- Network adapter 2: Host-only

**Resource warning:** Wazuh's all-in-one deployment recommends more memory than this VM has. The host laptop also has 8 GB RAM. The stack did start, but swap use and host freezes were observed. Avoid running another large VM at the same time. Consider upgrading host RAM before doing extended investigations.

### 1.2 Ubuntu Server VM — installed, not deployed

An Ubuntu Server 24.04.5 VM was installed and prepared, but **Wazuh was not installed or run on it**. We chose Kali Linux with Docker Compose for the actual deployment because that setup was already working and avoided adding another active VM to a host with only 8 GB of RAM. Ubuntu was left powered off to conserve resources.

**Possible future use:** Ubuntu Server can be considered as an alternative, dedicated host for the Wazuh stack if the lab is later rebuilt or migrated. That migration has not been performed or tested, so Ubuntu must not be described as the current Wazuh server. Do not run the Ubuntu VM alongside Kali on this machine while memory is limited.

## 2. Update Kali

Open a terminal in Kali and run:

```bash
sudo apt update
sudo apt full-upgrade -y
```

- `sudo` runs the command with administrator privileges.
- `apt update` refreshes the package index; it does not itself upgrade installed software.
- `apt full-upgrade -y` installs available upgrades and automatically confirms prompts.

If Kali asks for a reboot after a kernel update, reboot before continuing.

## 3. Install Docker, Git, and Docker Compose

Run:

```bash
sudo apt install -y docker.io git docker-compose
```

What this installs:
- `docker.io`: Docker Engine package, which runs containers.
- `git`: downloads and manages the Wazuh deployment repository.
- `docker-compose`: the standalone Compose command used in this Kali setup.

**Important for this machine:** the working command is `docker-compose` with a hyphen. The `docker compose` subcommand was not available.

Enable and start Docker:

```bash
sudo systemctl enable docker --now
```

- `systemctl` manages Linux services.
- `enable` configures Docker to start automatically at boot.
- `--now` starts the service immediately.

Check Docker:

```bash
sudo systemctl status docker
docker --version
docker-compose --version
git --version
```

The Docker service should show as active/running. The versions recorded in this lab were Docker 28.5.2, Docker Compose 2.40.3-3, and Git 2.53.0.

## 4. Configure the Linux kernel setting required by Wazuh Indexer

Run:

```bash
sudo sysctl -w vm.max_map_count=262144
```

This raises the maximum number of memory-mapped areas available to a process. OpenSearch, which Wazuh Indexer is based on, requires this setting.

Verify it:

```bash
sysctl vm.max_map_count
```

Expected value:

```text
vm.max_map_count = 262144
```

The command above changes the setting at runtime. It may reset after a reboot. If the Indexer fails after restarting Kali, check this setting and reapply it if necessary.

## 5. Download the official Wazuh Docker deployment

Run:

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.9
cd wazuh-docker/single-node
```

Explanation:
- `git clone` downloads the official Wazuh Docker deployment repository.
- `-b v4.14.9` selects the Wazuh release tag used for this lab.
- `cd` enters the single-node deployment directory.

List the files:

```bash
ls
```

The directory should contain files/folders such as `docker-compose.yml`, `generate-indexer-certs.yml`, `config`, and `README.md`.

## 6. Validate the Compose configuration

From `~/wazuh-docker/single-node`, run:

```bash
sudo docker-compose config --quiet
```

This asks Compose to parse and validate the YAML configuration without printing the full configuration. No output and no error means validation completed successfully.

## 7. Generate the certificates

Run:

```bash
sudo docker-compose -f generate-indexer-certs.yml run --rm generator
```

Explanation:
- `-f generate-indexer-certs.yml` tells Compose to use the certificate-generation configuration instead of the main stack file.
- `run` starts the one-off certificate generator container.
- `--rm` removes that temporary container after it exits.

The generator creates certificates used for encrypted communication between Wazuh components. In this lab, certificate generation completed and the expected certificate files were present.

**Security:** certificate files and private keys must remain private. Never upload the generated certificate directory, `.pem` files, or private keys to this public portfolio repository.

## 8. Start Wazuh

Run:

```bash
sudo docker-compose up -d
```

- `up` creates and starts the services described in `docker-compose.yml`.
- `-d` runs them in the background so the terminal is returned to you.

Check container status:

```bash
sudo docker ps -a
```

The deployment should contain three main services:
1. Wazuh Manager — receives endpoint data and performs analysis.
2. Wazuh Indexer — stores and indexes events and alerts.
3. Wazuh Dashboard — web interface for searching and investigating data.

In this lab, all three containers were observed in the Up state.

If a container is not running, inspect its logs:

```bash
sudo docker-compose logs --tail=100
```

To inspect one service, use its service name from the Compose file, for example:

```bash
sudo docker-compose logs --tail=100 wazuh.indexer
```

## 9. Verify Indexer health and resource use

Check recent Indexer logs:

```bash
sudo docker-compose logs --tail=100 wazuh.indexer
```

The successful startup log in this lab included the cluster health changing to GREEN. GREEN indicates the single-node Indexer cluster reached its healthy state.

Check Linux memory and disk:

```bash
free -h
df -h
```

Check live container resource use:

```bash
sudo docker stats --no-stream
```

These commands help identify memory pressure:
- `free -h` reports RAM and swap. Pay attention to **available**, not just the small **free** figure, because Linux uses spare RAM for cache.
- `df -h` reports filesystem capacity and free space.
- `docker stats` shows resource use by each container.

The Indexer used roughly 1.1 GiB in one observation. The Kali VM also used swap. These are observations from this machine, not fixed Wazuh requirements or performance guarantees.

## 10. Open the Wazuh Dashboard

Inside Kali, open a browser and visit:

```text
https://localhost
```

The browser may display a certificate warning because this is a locally generated lab certificate. Confirm that you are connecting to your own local lab endpoint before proceeding.

The default example credentials in the vendor configuration were left unchanged for this isolated lab at the user's request. Do not publish them in documentation or expose this dashboard outside the lab. Change all default credentials before any external exposure.

The dashboard successfully loaded in this lab.

## 11. Configure VirtualBox networking for the Windows endpoint

The Kali VM needs:
- NAT for internet access inside Kali.
- Host-only networking for private communication with the Windows host.

To change adapters, power off the Kali VM first. In VirtualBox:
1. Select the Kali VM.
2. Open **Settings → Network**.
3. Keep **Adapter 1** enabled and attached to **NAT**.
4. Enable **Adapter 2** and attach it to **Host-only Adapter**.
5. Save the settings and start Kali.

Find Kali's addresses:

```bash
ip -br address
```

The addresses observed in this lab were:
- `eth0`: `10.0.2.15/24` — NAT.
- `eth1`: `192.168.56.101/24` — Host-only.

The host-only address is the manager address Windows should use in this lab, provided it remains unchanged after a reboot.

## 12. Test connectivity from Windows

Open PowerShell on the Windows host.

Test basic reachability:

```powershell
ping 192.168.56.101
```

Test the Wazuh ports:

```powershell
Test-NetConnection 192.168.56.101 -Port 1514
Test-NetConnection 192.168.56.101 -Port 1515
Test-NetConnection 192.168.56.101 -Port 55000
```

Explanation:
- `Test-NetConnection` tests TCP connectivity to a host and port.
- Port 1514 is used for Wazuh agent communication.
- Port 1515 is used for agent enrollment.
- Port 55000 is the Wazuh API.

In this lab, all three tests returned `TcpTestSucceeded : True`. This proves the ports were reachable from Windows at the time of testing; it does **not** prove the agent has enrolled or is sending events.

## 13. Install the Wazuh agent on Windows

The downloaded installer was:

```text
wazuh-agent-4.14.9-1.msi
```

The installer was downloaded from the official Wazuh packages/download page and installed using the Windows graphical installer.

Recorded actions:
1. Open the downloaded MSI.
2. Follow the installer prompts.
3. Complete the installation.
4. Open PowerShell to inspect the service.

Check the service:

```powershell
Get-Service WazuhSvc
```

Start it if it is stopped:

```powershell
Start-Service Wazuh
```

The service was observed running after it was started. The service can also be addressed by its service name, `WazuhSvc`.

**Important pending check:** a running Windows service alone does not prove it is configured with the correct manager address. Verify the manager address in the agent configuration file and then confirm the endpoint appears in the Wazuh dashboard. Common install locations include `C:\Program Files (x86)\ossec-agent\ossec.conf` and `C:\Program Files\ossec-agent\ossec.conf`. First confirm which folder exists on this machine; do not assume the path.

## 14. Verify endpoint enrollment — pending

In the Wazuh Dashboard, open the endpoint/agent management page (the exact menu label can vary by dashboard version) and look for the Windows host.

Expected outcome:
- The Windows endpoint appears in the list.
- Its status becomes **Active**.

If it does not appear, do not mark the integration as complete. Check:
1. The manager address in the Windows agent configuration.
2. Whether the Windows agent service is running.
3. Agent logs on Windows.
4. Manager logs in Kali.
5. Whether enrollment port 1515 and agent communication port 1514 are reachable.

Useful Windows service check:

```powershell
Get-Service WazuhSvc
```

Useful Kali container log check:

```bash
sudo docker-compose logs --tail=100 wazuh.manager
```

We will add the exact configuration edits and log paths after inspecting the actual installation on this Windows machine, rather than guessing.

## 15. Verify event ingestion — pending

After the agent shows Active:
1. Open the endpoint details in Wazuh.
2. Confirm recent events are arriving.
3. Generate a safe, controlled test event on the Windows host.
4. Search for that event in the dashboard.
5. Record the event time, rule/alert details, and what the event means.

Do not claim detection success until the event is visible in Wazuh and the evidence has been checked.

## 16. Useful operating commands

Run these from the `single-node` directory.

Check containers:

```bash
sudo docker ps
```

Follow logs:

```bash
sudo docker-compose logs -f
```

Stop the stack:

```bash
sudo docker-compose down
```

Start it again:

```bash
sudo docker-compose up -d
```

The `down` command stops and removes the Compose containers and network, but normally preserves named volumes unless you add the destructive `-v` option. **Do not use `-v`** unless you intentionally want to remove stored lab data.

## 17. Current completion checklist

- [x] Kali updated and required packages installed
- [x] Docker service enabled and running
- [x] Kernel memory-map setting applied
- [x] Official Wazuh Docker repository cloned at v4.14.9
- [x] Compose configuration validated
- [x] Certificates generated
- [x] Wazuh Manager, Indexer, and Dashboard containers started
- [x] Indexer cluster reached GREEN
- [x] Dashboard opened locally
- [x] VirtualBox Host-only networking configured
- [x] Windows host reached Kali and tested required TCP ports
- [x] Windows Wazuh agent installed and service started
- [ ] Confirm Windows agent manager address
- [ ] Confirm Windows endpoint is Active in Wazuh
- [ ] Confirm Windows events are ingested
- [ ] Perform and document first alert investigation

## 18. References

- [Official Wazuh Docker deployment](https://github.com/wazuh/wazuh-docker)
- [Wazuh documentation](https://documentation.wazuh.com/)
- [Wazuh installation guide](https://documentation.wazuh.com/current/installation-guide/index.html)
