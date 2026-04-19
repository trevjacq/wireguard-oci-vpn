WireGuard VPN on Oracle Cloud Infrastructure (Always Free Tier)
A complete walkthrough for deploying a self-hosted WireGuard VPN server on Oracle Cloud Infrastructure's Always Free Tier. This project demonstrates cloud networking, Linux server administration, firewall troubleshooting, and VPN architecture — deployed at zero cost on OCI's permanently free compute resources.
Table of Contents

Project Overview
Architecture
Prerequisites
Phase 1: OCI Infrastructure Setup

Step 1: Create a Virtual Cloud Network (VCN)
Step 2: Provision the Compute Instance
Step 3: Connect via SSH


Phase 2: WireGuard Server Configuration

Step 4: Install WireGuard
Step 5: Generate Server and Client Keys
Step 6: Create the Server Configuration
Step 7: Enable IP Forwarding
Step 8: Fix OCI Default iptables Rules
Step 9: Start WireGuard


Phase 3: OCI Security List Configuration

Step 10: Open UDP Port 51820 in OCI Security List


Phase 4: Client Configuration

Step 11: Install WireGuard Client (Windows)
Step 12: Create the Client Tunnel Configuration
Step 13: Activate and Test


Troubleshooting

OCI iptables REJECT Rules (Critical)
Handshake Never Completes
Connected but No Internet
SSH Drops When VPN Activates


Security Considerations
Skills Demonstrated
Resources


Project Overview
This project deploys a full-tunnel WireGuard VPN server on Oracle Cloud Infrastructure using the Always Free Tier. When connected, all client internet traffic routes through the OCI server, masking the client's real IP address and encrypting all traffic between the client and the server.
Why WireGuard over OpenVPN?

Dramatically simpler configuration (single config file vs. certificates + keys + server configs)
Faster connection establishment (1 RTT handshake vs. multi-step TLS negotiation)
Better performance (kernel-space implementation, ~3x throughput vs. OpenVPN in benchmarks)
Smaller attack surface (~4,000 lines of code vs. ~100,000+ for OpenVPN)
Modern cryptography (Curve25519, ChaCha20, Poly1305, BLAKE2s)

What this project covers:

OCI cloud infrastructure provisioning (VCN, subnets, gateways, security lists, compute)
Linux server hardening and firewall configuration
WireGuard VPN architecture and deployment
Network troubleshooting with tcpdump, iptables, and ss
Diagnosing and resolving OCI-specific iptables issues that most tutorials miss


Architecture
┌─────────────────┐         UDP 51820         ┌──────────────────────────┐
│  Windows Client │◄─── WireGuard Tunnel ────►│  OCI Ubuntu VM           │
│  10.66.66.2/32  │     (Encrypted)           │  10.66.66.1/24 (wg0)    │
│                 │                           │  10.0.0.x/24 (ens3)     │
│  WireGuard App  │                           │  <public-ip> (NAT)      │
└─────────────────┘                           └──────────┬───────────────┘
                                                         │
                                                         │ iptables MASQUERADE
                                                         ▼
                                              ┌──────────────────────────┐
                                              │     Internet             │
                                              │  (sees OCI public IP)    │
                                              └──────────────────────────┘
Traffic flow:

Client sends all traffic into the WireGuard tunnel (encrypted with ChaCha20-Poly1305)
Server receives traffic on the wg0 interface (10.66.66.1)
iptables FORWARD rules allow traffic between wg0 and ens3
iptables NAT (MASQUERADE) rewrites the source IP to the server's public IP
Traffic exits to the internet from the OCI server
Responses follow the reverse path back through the tunnel to the client

OCI Network Stack:
Internet
    │
    ▼
┌─────────────────────────────┐
│  OCI Security List          │  ◄── Cloud-level firewall (Layer 3/4)
│  - TCP 22 (SSH)             │      Must explicitly allow UDP 51820
│  - UDP 51820 (WireGuard)    │
│  - ICMP                     │
└─────────────┬───────────────┘
              ▼
┌─────────────────────────────┐
│  Internet Gateway           │  ◄── Routes 0.0.0.0/0 to/from internet
└─────────────┬───────────────┘
              ▼
┌─────────────────────────────┐
│  Public Subnet (10.0.0.0/24)│  ◄── Instance lives here with public IP
└─────────────┬───────────────┘
              ▼
┌─────────────────────────────┐
│  Ubuntu VM (iptables)       │  ◄── OS-level firewall
│  - INPUT chain              │      OCI images have hidden REJECT rules
│  - FORWARD chain            │      that block VPN traffic by default
│  - NAT POSTROUTING          │
└─────────────────────────────┘

Prerequisites

Oracle Cloud Infrastructure account — sign up at cloud.oracle.com. The Always Free Tier includes 2 AMD Micro instances (VM.Standard.E2.1.Micro) permanently free.
Windows 10/11 (for the client) — macOS, Linux, iOS, and Android clients also work; this guide focuses on the Windows client.
Basic familiarity with SSH and the Linux command line.
A terminal emulator — Windows PowerShell works. Windows Terminal (from the Microsoft Store) is recommended for better paste handling.


Phase 1: OCI Infrastructure Setup
Step 1: Create a Virtual Cloud Network (VCN)
The VCN provides the network foundation for the compute instance. The VCN Wizard automatically creates all required components: public and private subnets, internet gateway, NAT gateway, service gateway, and properly configured route tables.

Important: Do not try to create a VCN inline from the "Create Instance" page. OCI's UI can become unresponsive when creating networking resources inline. Always create the VCN as a standalone step first.


In the OCI Console, navigate to Networking → Virtual Cloud Networks.
Ensure the compartment dropdown shows your root compartment.
Open the Actions ▼ dropdown and select Start VCN Wizard.

If you don't see this option, check for a "Start VCN Wizard" button elsewhere on the page — OCI periodically redesigns the UI.


Select "Create VCN with Internet Connectivity" → click Start VCN Wizard.
Configure:

VCN name: vpn-vcn
Compartment: your root compartment
VCN CIDR block: 10.0.0.0/16 (default)
Public Subnet CIDR: 10.0.0.0/24 (default)
Private Subnet CIDR: 10.0.1.0/24 (default)


Click Next, review the summary, then click Create.
Wait approximately 30 seconds for all resources to provision. Click View VCN when complete.

What the wizard creates:
ResourcePurposeVCN (10.0.0.0/16)Virtual network containerPublic Subnet (10.0.0.0/24)Internet-accessible subnet for the VMPrivate Subnet (10.0.1.0/24)Internal-only subnet (unused in this project)Internet GatewayRoutes traffic to/from the public internetNAT GatewayAllows private subnet outbound access (unused here)Service GatewayAccess to Oracle services (unused here)Route TablesDefault routes for each subnetSecurity ListsDefault firewall rules (SSH + ICMP allowed inbound)
Step 2: Provision the Compute Instance

Navigate to Compute → Instances → click Create Instance.
Configure:

Name: vpn-server (or any descriptive name)
Compartment: your root compartment
Image: Ubuntu 22.04 (Canonical)
Shape: VM.Standard.E2.1.Micro — look for the "Always Free-eligible" badge
Networking:

Select existing virtual cloud network → vpn-vcn
Select existing subnet → the Public Subnet (name includes "Public Subnet-vpn-vcn")
Automatically assign public IPv4 address: toggle ON


SSH keys: select "Generate a key pair for me"

Download both the private key (.key) and public key (.pub) files
Store them in a dedicated folder (e.g., C:\Users\<you>\oci-keys\)


Boot volume: leave defaults


Click Create and wait 1–2 minutes for the instance to reach Running state.
On the Instance Details page, note the Public IPv4 address — you'll need this for SSH and VPN client configuration.


SSH Key Security: The private key file is the sole authentication mechanism for your server. Treat it like a password. Don't store it in cloud-synced folders (OneDrive, Dropbox, Google Drive) where it gets replicated to third-party servers.

Step 3: Connect via SSH
Fix Windows file permissions (OpenSSH refuses keys with overly permissive access):
powershellcd C:\Users\<you>\oci-keys
icacls ssh-key-YYYY-MM-DD.key /inheritance:r
icacls ssh-key-YYYY-MM-DD.key /grant:r "$($env:USERNAME):(R)"
Connect:
powershellssh -i ssh-key-YYYY-MM-DD.key ubuntu@<public-ip>
Accept the host fingerprint on first connection by typing yes.
Update the system:
bashsudo apt update && sudo apt upgrade -y
sudo reboot
Reconnect after the reboot completes (~45 seconds).

Phase 2: WireGuard Server Configuration
Step 4: Install WireGuard
bashsudo apt install wireguard -y
which wg  # Should print /usr/bin/wg
Step 5: Generate Server and Client Keys
WireGuard uses Curve25519 key pairs. Each endpoint (server + each client) needs its own pair.
Server keys:
bashwg genkey | sudo tee /etc/wireguard/server_private.key
sudo cat /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key
sudo chmod 600 /etc/wireguard/server_private.key
Client keys (one pair per device):
bashwg genkey | sudo tee /etc/wireguard/client1_private.key
sudo cat /etc/wireguard/client1_private.key | wg pubkey | sudo tee /etc/wireguard/client1_public.key
sudo chmod 600 /etc/wireguard/client1_private.key
Record all four values — you'll need them for the configuration files:
bashsudo cat /etc/wireguard/server_private.key   # → SERVER_PRIVATE_KEY
sudo cat /etc/wireguard/server_public.key    # → SERVER_PUBLIC_KEY
sudo cat /etc/wireguard/client1_private.key  # → CLIENT_PRIVATE_KEY
sudo cat /etc/wireguard/client1_public.key   # → CLIENT_PUBLIC_KEY
Identify the network interface name:
baship -o -4 route show to default | awk '{print $5}'
This returns the name of the public-facing interface (typically ens3 on OCI). You'll need this for the NAT rules.
Step 6: Create the Server Configuration
Create /etc/wireguard/wg0.conf:
ini[Interface]
Address = 10.66.66.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

PostUp = iptables -I INPUT 1 -p udp --dport 51820 -j ACCEPT; iptables -I FORWARD 1 -i wg0 -j ACCEPT; iptables -I FORWARD 1 -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
PostDown = iptables -D INPUT -p udp --dport 51820 -j ACCEPT; iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE

[Peer]
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 10.66.66.2/32
Replace SERVER_PRIVATE_KEY and CLIENT_PUBLIC_KEY with actual key values. Replace ens3 with your interface name if different.
Lock down the config file:
bashsudo chmod 600 /etc/wireguard/wg0.conf
Configuration breakdown:
DirectivePurposeAddress = 10.66.66.1/24Server's IP inside the VPN tunnelListenPort = 51820UDP port WireGuard listens onPrivateKeyServer's private key (never shared)PostUp (INPUT rule)Allows incoming WireGuard handshake packetsPostUp (FORWARD rules)Allows forwarding between tunnel and internetPostUp (MASQUERADE)NAT — rewrites client source IPs to server's public IPPostDownReverses all PostUp rules when WireGuard stopsAllowedIPs = 10.66.66.2/32Only this client IP is permitted through the tunnel

Critical: Use -I FORWARD 1 (insert at top), NOT -A FORWARD (append to bottom). OCI's default Ubuntu image includes hidden iptables REJECT rules in both the INPUT and FORWARD chains. Appending rules places them AFTER the REJECT, where they'll never match. Inserting at position 1 ensures they're evaluated first. See Troubleshooting for full details.

Step 7: Enable IP Forwarding
Without IP forwarding enabled in the Linux kernel, packets won't be routed between the wg0 tunnel interface and the ens3 public interface — VPN clients would connect but have no internet access.
bashecho 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-wireguard-forward.conf
sudo sysctl -p /etc/sysctl.d/99-wireguard-forward.conf
Verify: output should show net.ipv4.ip_forward = 1.
Step 8: Fix OCI Default iptables Rules

This is the step most WireGuard-on-OCI tutorials miss. OCI's Ubuntu images ship with restrictive default iptables rules that silently block VPN traffic. If you skip this step, the VPN will appear to start correctly but no traffic will flow.

The problem:
OCI pre-configures iptables with REJECT-all rules at the end of both the INPUT and FORWARD chains:
Chain INPUT:
  1  ACCEPT  state RELATED,ESTABLISHED
  2  ACCEPT  icmp
  3  ACCEPT  loopback
  4  ACCEPT  tcp dpt:22 (SSH)
  5  REJECT  all  ← Blocks everything else, including UDP 51820

Chain FORWARD:
  1  REJECT  all  ← Blocks all forwarding (kills VPN routing)
The fix:
Insert ACCEPT rules at the TOP of each chain, before the REJECT rules:
bash# Allow WireGuard handshake packets (INPUT chain)
sudo iptables -I INPUT 1 -p udp --dport 51820 -j ACCEPT

# Allow forwarding between tunnel and internet (FORWARD chain)
sudo iptables -I FORWARD 1 -i wg0 -j ACCEPT
sudo iptables -I FORWARD 1 -o wg0 -j ACCEPT
The wg0.conf PostUp line already includes these commands (using -I not -A), so they'll be applied automatically whenever WireGuard starts. But you need to run them manually the first time, or restart WireGuard after creating the config.
Verify the fix:
bashsudo iptables -L INPUT -v -n --line-numbers
sudo iptables -L FORWARD -v -n --line-numbers
ACCEPT rules for UDP 51820 and wg0 should appear ABOVE any REJECT rules.
Step 9: Start WireGuard
bashsudo systemctl enable --now wg-quick@wg0
sudo systemctl status wg-quick@wg0
You should see:

Active: active (exited) in green — "exited" is normal; wg-quick runs setup commands and hands off to the kernel
status=0/SUCCESS

Verify the tunnel interface is up:
bashsudo wg
Expected output:
interface: wg0
  public key: <SERVER_PUBLIC_KEY>
  private key: (hidden)
  listening port: 51820

peer: <CLIENT_PUBLIC_KEY>
  allowed ips: 10.66.66.2/32

Note: The "latest handshake" and "transfer" lines won't appear until a client connects. Their absence at this stage is expected.


Phase 3: OCI Security List Configuration
Step 10: Open UDP Port 51820 in OCI Security List
The OCI Security List is a cloud-level firewall that operates independently from iptables on the VM. Even if iptables allows UDP 51820, OCI will block it at the network edge unless an explicit ingress rule exists.

Navigate to Networking → Virtual Cloud Networks → vpn-vcn.
Click the Security tab (or Subnets → Public Subnet → Security Lists).
Click Default Security List for vpn-vcn.
Click the Security rules tab → Add Ingress Rules.
Configure:

Stateless: unchecked
Source Type: CIDR
Source CIDR: 0.0.0.0/0
IP Protocol: UDP (not TCP — this is the most common mistake)
Source Port Range: (leave blank)
Destination Port Range: 51820
Description: WireGuard VPN


Click Add Ingress Rules.

Verify the rule appears in the Ingress Rules table:
StatelessSourceIP ProtocolDestination Port RangeAllowsNo0.0.0.0/0UDP51820UDP traffic for ports: 51820
Also confirm the Egress Rules section has a default "All Protocols to 0.0.0.0/0" rule — this allows the server to send responses back to clients. The VCN Wizard creates this by default.

Phase 4: Client Configuration
Step 11: Install WireGuard Client (Windows)

Download the Windows installer from wireguard.com/install.
Run the .msi installer (quick, no options needed).
Launch WireGuard from the Start menu.

WireGuard clients are also available for macOS, Linux, iOS, and Android.
Step 12: Create the Client Tunnel Configuration

In the WireGuard app, click the dropdown arrow next to "Add Tunnel" → "Add empty tunnel..."
Delete all auto-generated content in the text box.
Name: oci-vpn (no spaces — the name becomes a network interface name).
Paste this configuration:

ini[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.66.66.2/32
DNS = 1.1.1.1, 1.0.0.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = <SERVER_PUBLIC_IP>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
Replace CLIENT_PRIVATE_KEY, SERVER_PUBLIC_KEY, and <SERVER_PUBLIC_IP> with your actual values.

Important: The SERVER_PUBLIC_KEY must match what WireGuard is actually using on the server. Verify with sudo wg on the server — the public key shown in the interface: section is the authoritative value. This may differ from the contents of server_public.key if keys were regenerated.

Configuration breakdown:
DirectivePurposePrivateKeyClient's private key (never leaves the client device)Address = 10.66.66.2/32Client's IP inside the VPN tunnelDNS = 1.1.1.1, 1.0.0.1Cloudflare DNS — ensures DNS queries go through the tunnelPublicKeyServer's public key (for authenticating the server)EndpointServer's public IP and WireGuard portAllowedIPs = 0.0.0.0/0Route ALL traffic through the VPN (full tunnel)PersistentKeepalive = 25Sends a keepalive every 25s to maintain NAT state

Click Save.
Verify the "Public key" field at the top shows the expected client public key.

Step 13: Activate and Test

Select the oci-vpn tunnel in the WireGuard app.
Click Activate.
Status should flip to green "Active" within 5 seconds.
The Peer section should show:

Latest handshake: a few seconds ago
Transfer: non-zero bytes in both directions



Verification:
Open a browser and navigate to https://ifconfig.me or https://www.whatismyip.com. Your IP address should display as your OCI server's public IP, not your real home IP.
On the server, verify the handshake:
bashsudo wg
The peer section should now show latest handshake and transfer data.

Troubleshooting
OCI iptables REJECT Rules (Critical)
Symptom: WireGuard starts successfully on the server (sudo wg shows the interface and peer), the OCI Security List allows UDP 51820, but the client never completes a handshake. The client shows "Active" but no "Latest handshake" appears, and the Transfer line shows sent bytes but zero received.
Root Cause: OCI's default Ubuntu image includes hidden iptables REJECT rules at the end of the INPUT and FORWARD chains. These rules reject all traffic that doesn't match an earlier ACCEPT rule. Since there are no default rules for UDP 51820 or for forwarded traffic, WireGuard handshake packets and routed tunnel traffic are silently rejected.
Diagnosis:
bash# Check INPUT chain — look for REJECT at the bottom
sudo iptables -L INPUT -v -n --line-numbers

# Check FORWARD chain — look for REJECT at the top or bottom
sudo iptables -L FORWARD -v -n --line-numbers
Fix: Use -I (insert) instead of -A (append) in your wg0.conf PostUp rules, ensuring ACCEPT rules are placed BEFORE the REJECT rules. See Step 6 and Step 8 for the correct configuration.
Verification with tcpdump:
bash# Watch for incoming WireGuard handshake packets
sudo tcpdump -i ens3 -n udp port 51820
If packets appear but WireGuard doesn't process them, the INPUT chain is blocking them. If WireGuard processes them but forwarded traffic doesn't flow, the FORWARD chain is blocking.
Handshake Never Completes
Symptoms: Client shows "Active" but no "Latest handshake" line appears.
Check:

Keys match? Client config's [Peer] PublicKey must equal the server's actual public key (from sudo wg, not from the key file). Server config's [Peer] PublicKey must equal the client's public key.
UDP 51820 open in OCI Security List? Double-check the protocol is UDP, not TCP.
Packets reaching the server? Run sudo tcpdump -i ens3 -n udp port 51820 on the server while the client attempts to connect.
INPUT chain allowing packets? Run sudo iptables -L INPUT -v -n --line-numbers and verify an ACCEPT rule for UDP 51820 exists before any REJECT rule.

Connected but No Internet
Symptoms: Handshake succeeds (client shows "Latest handshake: X seconds ago") but websites don't load.
Check:

IP forwarding enabled? cat /proc/sys/net/ipv4/ip_forward should return 1.
FORWARD chain rules in correct position? sudo iptables -L FORWARD -v -n --line-numbers — ACCEPT rules for wg0 must be before any REJECT rule.
MASQUERADE rule exists? sudo iptables -t nat -L POSTROUTING -v -n should show a MASQUERADE rule on ens3.
Egress Security List rule? Verify OCI's Egress Rules include "All Protocols to 0.0.0.0/0".

SSH Drops When VPN Activates
Cause: The client config uses AllowedIPs = 0.0.0.0/0, which routes ALL traffic through the tunnel — including SSH traffic to the server. When the tunnel isn't passing traffic correctly, SSH breaks.
Workaround: Accept that SSH will drop during testing. Deactivate the VPN to restore connectivity, then SSH back in.
Permanent fix (optional): Exclude the server's own IP from the tunnel by splitting AllowedIPs:
iniAllowedIPs = 0.0.0.0/0, ::/0
# Replace with:
AllowedIPs = 0.0.0.0/1, 128.0.0.0/1
This routes all traffic through the tunnel using two half-ranges that cover the full IPv4 space but have a more specific prefix than the default route, so the direct route to the server's IP (used by SSH) is preserved.

Security Considerations

Private keys never leave their device. The server's private key stays on the server; the client's private key stays on the client. Only public keys are exchanged.
Key rotation: Generate new client key pairs periodically and update both server and client configs.
Restrict AllowedIPs on the server: Each peer's AllowedIPs should be as narrow as possible (/32 for a single client) to prevent IP spoofing within the tunnel.
SSH key storage: Keep SSH private keys in local-only directories, not in cloud-synced folders (OneDrive, Dropbox).
OCI Security List hardening: Consider restricting the UDP 51820 source CIDR to your home IP range instead of 0.0.0.0/0 if you have a static or semi-static IP.
WireGuard does not provide anonymity. The OCI server's IP is traceable to your OCI account. WireGuard encrypts traffic between client and server, but traffic from server to destination is unencrypted (unless the destination itself uses HTTPS/TLS).


Skills Demonstrated

Cloud Infrastructure: OCI VCN provisioning, subnet architecture, internet gateway configuration, security list management, compute instance lifecycle
Linux Administration: Ubuntu server management, SSH key-based authentication, package management, systemd service management, sysctl kernel parameter tuning
Networking: IP forwarding, NAT (MASQUERADE), iptables firewall chain management (INPUT, FORWARD, POSTROUTING), UDP port management, network interface identification
VPN Architecture: WireGuard protocol deployment, Curve25519 key pair generation, full-tunnel routing, DNS leak prevention, client/server configuration
Security Operations: Firewall rule ordering and troubleshooting, tcpdump packet capture and analysis, defense-in-depth (cloud firewall + OS firewall), principle of least privilege in key management
Troubleshooting Methodology: Systematic layer-by-layer diagnosis (cloud firewall → OS firewall → application), packet capture to isolate failure points, identifying undocumented platform-specific behaviors (OCI iptables defaults)


Resources

WireGuard Official Documentation
Oracle Cloud Infrastructure Always Free Tier
OCI VCN and Subnet Overview
OCI Security Lists Documentation
WireGuard Cryptography Whitepaper


Author
Trevor Young — GitHub | LinkedIn
