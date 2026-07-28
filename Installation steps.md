# Create the VM in Proxmox
- Create VM → OS: Ubuntu 24.04.2 ISO
- System defaults
- Disk (VirtIO SCSI)
- CPU (2 cores)
- Memory (min 4GB)
- Network (bridge `vmbr---`)

# Install Ubuntu Server
## Standard install flow
- Network: configured static IP manually during install
- Subnet: `1x.xx.xx.0/24
- Address: `1xx.xx.xx.xx`
- Gateway/DNS: per network config
- SSH Setup: OpenSSH server installed (for remote management)
- Featured Server Snaps: skipped entirely (not needed for UISP)

# Install UISP (official method)
```
curl -fsSL https://uisp.ui.com/install > /tmp/uisp_inst.sh && sudo bash /tmp/uisp_inst.sh
```
<img width="1077" height="852" alt="Screenshot 2026-07-28 080139" src="https://github.com/user-attachments/assets/1cf0e90a-551a-4333-af4c-58865f60ccd9" />

⚠️ This is Ubiquiti's official installer. 
It:
- Installs Docker + Docker Compose
- Creates the `unms` system user and `/home/unms` data directory
- Pulls all required container images
- Generates `docker-compose.yml` and starts the stack
  
⚠️ Do **not** use unofficial GitHub mirrors for the install script — the only verified official source is `uisp.ui.com/install`.

# Fix: `unms-api` unhealthy / containers failing to start
- Symptom: `unms-api` container reports unhealthy, `unms-device-ws-1` fails to start with `dependency failed to start: container unms-api is unhealthy`.
- Cause: `vm.overcommit_memory` kernel setting was `0` (default), which UISP's backend services (Postgres/RabbitMQ-adjacent memory allocation) expect to be `1`.
<img width="1488" height="987" alt="Screenshot 2026-07-28 080639" src="https://github.com/user-attachments/assets/e33c87bb-e86b-47ce-87f9-3e91cad75d1b" />

# Fix:
```
sudo sysctl vm.overcommit_memory=1
echo "vm.overcommit_memory=1" | sudo tee -a /etc/sysctl.conf
```
- Then restart the stack:
```
sudo -i
cd /home/unms
docker compose down
docker compose up -d
```
- If any container is stuck in `Created` state and never starts (as happened with `unms-device-ws-1`), start it manually:
```
docker start unms-device-ws-1
```

# Verify all containers are healthy
```
docker ps -a
```
Expected containers, all `Up` (with `unms-api` and `unms-siridb` showing `healthy`):
- unms-api`
- unms-device-ws-1`
- ucrm`
- unms-netflow`
- unms-siridb`
- unms-nginx`
- unms-postgres`
- unms-rabbitmq`
- unms-fluentd`

# First-run setup
- Navigate to `https://1xx.xx.xx.xx` in a browser:
- Accept the self-signed certificate warning
- Complete the UISP setup wizard: create admin account, organization name, optional SMTP config
- Land on the UISP dashboard


