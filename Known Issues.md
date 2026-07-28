# Known Issues
- Low free memory at idle (~157MB free on 3.8GB total). The stack runs, but there's little headroom. Recommend increasing VM memory to 6144–8192 MB via Proxmox: Shutdown VM → Hardware → Memory → increase → Start.
- Ubuntu 24.04 + `docker-compose-plugin`: some environments report apt can't find this package. If the installer fails on this, run:
```
  sudo apt install docker.io docker-compose docker-compose-plugin -y
  ```
- then re-run the UISP install script.

⚠️ Docker version warning: installer may warn the installed Docker version is "below recommended" and offer to auto-update — safe to decline (`N`) if the stack is otherwise healthy.
