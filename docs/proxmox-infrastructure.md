# Proxmox Infrastructure — Networking Setup & Troubleshooting

**Environment:** B2C-DevOps-Inter-Bootcamp-MAY26-English-group1 (Proxmox VE 8.4.20, hosted on Scaleway, public IP 51.159.98.38), assigned as a dedicated development environment.

This document records a real infrastructure incident: VM 103 (`farhad-eshop-k3s`) had no internet access after creation. What looked like a single-VM problem turned out to be a host-level bug affecting every VM on the node. This serves both as a record of the fix and as a runbook if networking breaks again (relevant to the project's Disaster Recovery requirements).

---

## Diagnosis

### The network topology
- `vmbr0` — public bridge, bound to the physical NIC, holds the host's public IP. **Host uplink only — not for VM traffic.**
- `vmbr1` — private virtual bridge, `10.10.10.1/24`, no physical NIC. **This is the bridge VMs should use.**

VM 103 was originally created on `vmbr0` by mistake (a habit worth avoiding — every other VM on this node used `vmbr1`).

### Root cause, found step by step
1. Confirmed the **host itself** had full internet access (`ping -c 3 1.1.1.1` → 0% loss) — ruled out a whole-node outage.
2. Confirmed `net.ipv4.ip_forward = 1` (already enabled) and a `MASQUERADE` NAT rule already existed live in `iptables` — but it was **not persisted** in `/etc/network/interfaces`, so it would vanish on reboot.
3. Confirmed VM 103's own config was correct (`bridge=vmbr1`, no firewall flag) once moved off `vmbr0`.
4. Confirmed via `tcpdump -i vmbr1 -n port 67 or port 68` that the VM's DHCP request **was** reaching the bridge — proving the VM/bridge/network chain itself was healthy.
5. Found `dnsmasq.service` **did not exist on the host at all** — no DHCP server was ever running to answer requests on `vmbr1`.

**Root cause: no DHCP server on `vmbr1`, and the NAT rule wasn't persisted across reboots.** Neither the VM's configuration nor its OS network settings were ever the problem.

---

## Fix 1 — Persistent NAT + IP forwarding

Backed up existing config first:
```bash
mkdir -p /root/net-backup
cp /etc/network/interfaces /root/net-backup/interfaces.bak-$(date +%F-%H%M)
iptables-save > /root/net-backup/iptables.bak-$(date +%F-%H%M)
```

Rewrote `/etc/network/interfaces` so forwarding and NAT are applied automatically via `post-up`/`post-down` hooks on `vmbr1`, rather than relying on manually-run commands that don't survive a reboot: