Proxmox Infrastructure — Networking Setup & Troubleshooting
Environment: B2C-DevOps-Inter-Bootcamp-MAY26-English-group1 (Proxmox VE 8.4.20, hosted on Scaleway, public IP 51.159.98.38), assigned as a dedicated development environment.

This document records a real infrastructure incident: VM 103 (farhad-eshop-k3s) had no internet access after creation. What looked like a single-VM problem turned out to be a host-level bug affecting every VM on the node. This serves both as a record of the fix and as a runbook if networking breaks again (relevant to the project's Disaster Recovery requirements).

Diagnosis
The network topology
vmbr0 — public bridge, bound to the physical NIC, holds the host's public IP. Host uplink only — not for VM traffic.

vmbr1 — private virtual bridge, 10.10.10.1/24, no physical NIC. This is the bridge VMs should use.

VM 103 was originally created on vmbr0 by mistake (a habit worth avoiding — every other VM on this node used vmbr1).

Root cause, found step by step
Confirmed the host itself had full internet access (ping -c 3 1.1.1.1 → 0% loss) — ruled out a whole-node outage.

Confirmed net.ipv4.ip_forward = 1 (already enabled) and a MASQUERADE NAT rule already existed live in iptables — but it was not persisted in /etc/network/interfaces, so it would vanish on reboot.

Confirmed VM 103's own config was correct (bridge=vmbr1, no firewall flag) once moved off vmbr0.

Confirmed via tcpdump -i vmbr1 -n port 67 or port 68 that the VM's DHCP request was reaching the bridge — proving the VM/bridge/network chain itself was healthy.

Found dnsmasq.service did not exist on the host at all — no DHCP server was ever running to answer requests on vmbr1.

Root cause: no DHCP server on vmbr1, and the NAT rule wasn't persisted across reboots. Neither the VM's configuration nor its OS network settings were ever the problem.

Fix 1 — Persistent NAT + IP forwarding
Backed up existing config first:

bash
mkdir -p /root/net-backup
cp /etc/network/interfaces /root/net-backup/interfaces.bak-$(date +%F-%H%M)
iptables-save > /root/net-backup/iptables.bak-$(date +%F-%H%M)
Rewrote /etc/network/interfaces so forwarding and NAT are applied automatically via post-up/post-down hooks on vmbr1, rather than relying on manually-run commands that don't survive a reboot:

text
auto vmbr1
iface vmbr1 inet static
    address 10.10.10.1/24
    bridge_ports none
    bridge_stp off
    bridge_fd 0
    post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
    post-up   iptables -t nat -A POSTROUTING -s '10.10.10.0/24' -o vmbr0 -j MASQUERADE
    post-down iptables -t nat -D POSTROUTING -s '10.10.10.0/24' -o vmbr0 -j MASQUERADE
Applied without a disruptive reboot:

bash
ifquery --check vmbr0
ifquery --check vmbr1
ifreload -a
Removed the old duplicate manual rule (left over from before persistence was added):

bash
iptables -t nat -D POSTROUTING -s '10.10.10.0/24' -o vmbr0 -j MASQUERADE
Then proved persistence with a real reboot — confirmed forwarding and the NAT rule both came back automatically with zero manual steps:

bash
reboot
# after reboot:
sysctl net.ipv4.ip_forward
iptables -t nat -L -n -v
Fix 2 — DHCP server (dnsmasq, scoped to vmbr1 only)
bash
apt update && apt install dnsmasq
First attempt failed:

text
failed to create listening socket for port 53: Address already in use
This was a conflict with systemd-resolved, which already listens on 127.0.0.53:53. Fixed by scoping dnsmasq explicitly to vmbr1 only, so it never touches the address systemd-resolved owns:

bash
cat > /etc/dnsmasq.d/vmbr1.conf << 'EOF'
interface=vmbr1
bind-interfaces
except-interface=lo
no-dhcp-interface=vmbr0
listen-address=10.10.10.1
dhcp-range=10.10.10.100,10.10.10.200,255.255.255.0,12h
dhcp-option=3,10.10.10.1
dhcp-option=6,51.159.47.26,51.159.47.28
EOF
systemctl restart dnsmasq
Verified with:

bash
ss -tulpn | grep :53
Result: dnsmasq listening only on 10.10.10.1:53 (TCP+UDP), systemd-resolved untouched on 127.0.0.53/127.0.0.54 — no conflict.

Reserved a permanent IP for VM 103 (avoids address drift on future leases):

bash
echo "dhcp-host=bc:24:11:2d:99:67,10.10.10.198" >> /etc/dnsmasq.d/vmbr1.conf
systemctl restart dnsmasq
VM-side verification
bash
ip a                    # -> inet 10.10.10.198/24, dynamic, ens18
ping -c 3 10.10.10.1    # -> 0% packet loss (gateway)
ping -c 3 1.1.1.1       # -> 0% packet loss (internet)
curl -s ifconfig.me     # -> 51.159.98.38 (host's public IP — confirms NAT working end-to-end)
All confirmed working. sudo networkctl renew ens18 was used to force a fresh lease and confirm the MAC-based reservation held (Ubuntu 24.04 has no dhclient — networkctl renew <iface> is the modern equivalent, or systemctl restart systemd-networkd as a fallback).

SSH access (passwordless, via key)
On VM 103:

bash
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
A dedicated key pair (id_ed25519_proxmox) was generated locally and added to both the Proxmox host (root, needed for ProxyJump) and VM 103 (farhad), since the VM's private IP (10.10.10.198) is only reachable through the host, not directly from outside.

Local SSH config shortcut (~/.ssh/config):

text
Host 51.159.98.38
    User root
    IdentityFile ~/.ssh/id_ed25519_proxmox
    IdentitiesOnly yes

Host eshop-k3s
    HostName 10.10.10.198
    User farhad
    ProxyJump root@51.159.98.38
    IdentityFile ~/.ssh/id_ed25519_proxmox
    IdentitiesOnly yes
Result: ssh eshop-k3s connects directly, passwordlessly, through the host as a jump box.

Checklist — adding a new VM to this network
The host-level fixes above (persistent forwarding, persistent NAT, dnsmasq) are permanent and apply to every future VM automatically — none of the fixes above need repeating. For any new VM:

Set network bridge to vmbr1 (never vmbr0 — that has no DHCP and isn't meant for guest traffic).

Boot it, confirm ip a shows a 10.10.10.1xx address within seconds.

(Optional, recommended for long-running VMs) Reserve a fixed IP:

bash
qm config <new-vmid>   # get its MAC address
echo "dhcp-host=<mac>,10.10.10.<ip>" >> /etc/dnsmasq.d/vmbr1.conf
systemctl restart dnsmasq
Install SSH, reuse the existing key (no need to generate a new one per VM):

bash
sudo apt install -y openssh-server && sudo systemctl enable --now ssh
Add an SSH config shortcut following the eshop-k3s pattern above.

Lessons Learned
NAT rules added via raw iptables commands do not survive a reboot unless baked into /etc/network/interfaces via post-up/post-down.

dnsmasq conflicts with systemd-resolved on port 53 unless explicitly scoped with bind-interfaces and a specific listen-address.

Proxmox's SDN GUI entries can exist in the interface without being actually applied — always verify the underlying /etc/pve/sdn/*.cfg files exist on disk rather than trusting the GUI listing alone.

Ubuntu 24.04 has no dhclient — use networkctl renew <iface> instead.

Piping data into ssh hijacks stdin, breaking interactive host-key confirmation. Accept host keys interactively first, then embed data directly in the remote command rather than piping.

ssh only auto-tries default-named key files (id_rsa, id_ecdsa, id_ed25519) — a custom-named key needs -i <path> or an explicit IdentityFile in ~/.ssh/config.

Proxmox's browser Shell uses its own terminal proxy (termproxy), not real sshd — root access there does not guarantee root can SSH in over port 22; check sshd_config's PermitRootLogin separately.