# Maintenance
- Data location: `/home/unms/data` (device backups, firmware, config, timeseries data)
- Compose file: `/home/unms/docker-compose.yml`
- Restart stack: `cd /home/unms && docker compose restart`
- View logs for a container: `docker logs <container-name> --tail 50`
⚠️ Recommended: take a Proxmox VM snapshot now that the deployment is stable, and add this VM to the existing Proxmox backup schedule (`main-backup` / `vmbackup` storage, matching other VMs on this host).
