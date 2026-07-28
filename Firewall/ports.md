# Firewall / Ports

## Confirm the following are reachable from the management network:
- 443/tcp — UISP web UI (required)
- 80/tcp — HTTP redirect
- 2055/udp — NetFlow (if used)
- 8089/tcp — additional UISP service port

⚠️ If using Proxmox firewall on the NIC (`firewall=1`), ensure rules permit inbound 443 from the management subnet. On the guest itself:
```
sudo ufw allow 443/tcp
```
