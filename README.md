# NetSec Lab: Automated Reverse Proxy and SSH Lab

A small network security lab built with **VirtualBox**, **Vagrant**, **Debian 12**, **Nginx**, and **Tailscale**, driven from a Windows host. A single `vagrant up` builds a Debian VM that runs a reverse proxy and accepts key-based SSH logins, with no manual VM setup.

## Goals

1. **Reverse proxy:** Nginx re-routes traffic from port `8080` to a backend service on port `3000` that is not exposed outside the VM.
2. **Automation:** a `Vagrantfile` builds and configures the whole VM from code, so it can be destroyed and rebuilt at any time.
3. **Remote SSH access:** the host connects to the VM with SSH keys (no password), locally through a port forward and remotely over Tailscale.

## Architecture

```
Windows host                          Debian 12 VM (VirtualBox)
-----------------                     ------------------------------------
curl 127.0.0.1:8080  ──forward──►     Nginx :8080  ──proxy_pass──►  Python backend 127.0.0.1:3000
ssh -p 2222 ...      ──forward──►     sshd :22
Tailscale (100.x)  ◄──── tailnet ────►  Tailscale (100.x)   (sshd :22, Nginx :8080)
```

- Port forwards are bound to `127.0.0.1` on the host, so other devices on your LAN cannot reach the VM through them.
- The backend listens only on the VM's loopback interface. The only way to reach it is through the proxy.

## Prerequisites

- Windows 10/11 with hardware virtualization enabled in BIOS/UEFI
- [VirtualBox](https://www.virtualbox.org/)
- [Vagrant](https://developer.hashicorp.com/vagrant) (a version that supports your installed VirtualBox version)
- [Tailscale](https://tailscale.com/) account and client (needed for the remote-access phase)
- OpenSSH client and an ed25519 key pair at `%USERPROFILE%\.ssh\id_ed25519`

Install with winget (from an administrator PowerShell):

```powershell
winget install Oracle.VirtualBox
winget install Hashicorp.Vagrant
winget install Tailscale.Tailscale
```

> **Version note:** Vagrant must support your VirtualBox release. If `vagrant up` reports an unsupported VirtualBox version, update Vagrant or install a VirtualBox version it lists.

## Project files

| File | Purpose |
|------|---------|
| `Vagrantfile` | Defines the VM and provisioning steps |
| `.editorconfig` | Forces LF line endings so shell scripts work on Linux |
| `.gitignore` | Keeps local state and secrets out of Git |
| `README.md` | This file |

Suggested `.gitignore`:

```
.vagrant/
*.log
```

## Quick start

```powershell
git clone <your-repo-url> netsec-lab
cd netsec-lab
vagrant validate
vagrant up --provider=virtualbox
```

The first run downloads the Debian box, so it takes a few minutes.

## Verify the reverse proxy

Use `curl.exe` (plain `curl` in PowerShell is an alias for another command):

```powershell
curl.exe -i http://127.0.0.1:8080    # 200 OK, Server: nginx, backend message
curl.exe http://127.0.0.1:3000       # should fail: port 3000 is not exposed
```

The second command failing is the intended result: the backend is reachable only through Nginx.

## SSH access

Local access through the forwarded port:

```powershell
ssh -i $env:USERPROFILE\.ssh\id_ed25519 -p 2222 vagrant@127.0.0.1
```

After rebuilding the VM, SSH warns that the host key changed. Clear the old entry:

```powershell
ssh-keygen -R "[127.0.0.1]:2222"
```

Run a single command without opening a shell:

```powershell
ssh -i $env:USERPROFILE\.ssh\id_ed25519 -p 2222 vagrant@127.0.0.1 "systemctl status nginx --no-pager"
```

## Remote access with Tailscale

The Tailscale auth key is a credential. **Never write it into the Vagrantfile or commit it.** Pass it through an environment variable for the current PowerShell session only:

```powershell
$env:TS_AUTHKEY = "tskey-auth-xxxxxxxx"
vagrant up --provision
```

Once the VM joins your tailnet:

```powershell
ssh vagrant@netsec-lab
curl.exe http://netsec-lab:8080
```

MagicDNS must be enabled in the Tailscale admin console for the name to resolve. Otherwise, use the VM's `100.x.y.z` address.

## Everyday commands

| Command | Effect |
|---------|--------|
| `vagrant up` | Create or start the VM (provisioners run only on first creation) |
| `vagrant provision` | Re-run provisioners on a running VM |
| `vagrant halt` | Shut the VM down, keep its disk |
| `vagrant destroy -f` | Delete the VM entirely |
| `vagrant port` | Show active port forwards |

## Security notes

- Port forwards bind to `127.0.0.1` only.
- The backend binds to loopback inside the VM and is reachable only via Nginx.
- SSH uses key authentication. Password logins are disabled in the SSH phase.
- Secrets (Tailscale auth keys, private SSH keys) stay out of the repository.
- This is a lab environment. It is not hardened for production use.

## Troubleshooting

| Symptom | Likely cause and fix |
|---------|----------------------|
| `vagrant up` hangs at "Waiting for machine to boot" | Virtualization disabled in firmware, or Hyper-V/WSL2 conflict |
| Unsupported VirtualBox version error | Vagrant and VirtualBox versions are mismatched |
| `502 Bad Gateway` on port 8080 | Backend not running: check `systemctl status demo-backend` in the VM |
| Script errors mentioning `\r` | CRLF line endings: re-save files as LF |
| Provisioning didn't run | Provisioners only run on first creation: use `vagrant provision` |
| Port already in use | Another program holds the port: check `vagrant port` and change the host port |
| Tailscale name doesn't resolve | Enable MagicDNS or use the `100.x.y.z` address |

## Project status

- [x] Phase 0: Host preparation
- [x] Phase 1: Vagrantfile and base VM
- [x] Phase 2: Nginx reverse proxy (8080 → 3000)
- [x] Phase 3: Key-based SSH
- [x] Phase 4: Tailscale in the VM
- [x] Phase 5: Remote access from the host over Tailscale

> Update this checklist, and re-test the sections above, as you complete each phase.

## Ideas for extending the lab

- TLS on the reverse proxy
- Tailscale ACLs to restrict which devices can reach the VM
- A host firewall in the VM (nftables)

## License

Add a license of your choice (for example MIT).
