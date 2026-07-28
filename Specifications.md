# VM Specifications (Proxmox)

- OS: Ubuntu Server 24.04.2 LTS
- CPU: 2 cores (adjust upward if device count grows)
- Memory: 3.8 GiB (recommend bumping to 6–8 GiB — see Known Issues.md)
- Disk: VirtIO SCSI, sized per storage availability (recommend 40–60GB minimum for UISP's growing device backups/firmware/timeseries data)
- Network: VirtIO NIC on `vmbr---`, static IP config
- Guest OS profile: Linux, 6.x - 2.6 Kernel
