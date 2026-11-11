# AnsibleFlix — Managing the VM Lifecycle

## Stopping and Starting the VM

If you put your Mac to sleep, the Multipass VM will usually pause and resume automatically. However, if you restart or shut down your Mac, the VM will stop.

### Stop the VM cleanly (end of day)
```bash
multipass stop ansibleflix
```

### Start the VM again (resuming work)
```bash
multipass start ansibleflix
```

---

## ⚠️ Important: Check the IP Address When Resuming

When a Multipass VM restarts, it may be assigned a new local IP address by the internal DHCP server.

Before running Ansible tasks, check the current IP:
```bash
multipass info ansibleflix | grep IPv4
```

- If the IP is unchanged, you are ready to go.
- If the IP has changed, update `ansible_host` in `inventory/hosts.ini`:

```ini
[media_servers]
ansibleflix ansible_host=<NEW_IP>
```

---

## Data Persistence

All changes made inside the VM (SSH keys, installed packages, files, Docker containers) are stored on the VM's virtual disk.

As long as you do **not** run `multipass delete` or `multipass purge`, your VM state and data will remain intact across reboots.
