# StrongSwan IPsec VPN Setup for AWS EC2 (Site‑to‑Site with NBC Bank)

> **Note:** This documentation covers the **NBC Bank** setup first (end‑to‑end, as deployed and working in PROD).  
> **Part 2** at the end shows how to **add a second partner bank — Stanbic Bank** — without disturbing the working NBC tunnel.

## Overview

This document provides a step‑by‑step guide to replicate the IPsec IKEv2 VPN tunnel between an **AWS EC2 Ubuntu instance** (running StrongSwan) and the **FortiGate firewall** at NBC Bank. The setup is identical for both **UAT** and **PROD** environments; you only need to adjust the IP addresses.

### Architecture Summary

- The EC2 instance has a **private IP** (e.g., `10.0.4.83`) and a **public IP** (e.g., `63.178.83.38`) via AWS NAT.
- StrongSwan binds to the private IP, but presents the public IP as its identity and traffic selector.
- NBC’s FortiGate uses the public IP as the remote address in Phase 2.
- A loopback alias on the server ensures the kernel accepts packets destined to the public IP.

---

## Prerequisites

- An AWS EC2 instance running **Ubuntu 22.04/24.04/26.04**.
- **SSH access** with `sudo` privileges.
- **Public IP** of the instance (provided by AWS – Elastic IP or auto‑assigned).
- **Private IP** of the instance (visible via `ip addr` or `ifconfig`).
- Information from NBC:
  - FortiGate public IP: `102.212.82.5`
  - Their internal tunnel ID: `10.100.0.17`
  - Protected subnets on NBC side: `196.45.159.14/32` and `196.45.159.17/32`
  - Pre‑shared key (PSK) – exchanged securely.
- IKE parameters: IKEv2, AES256, SHA256, DH Group 20 (ECP384).

---

## Step 1: Update System and Install StrongSwan

Connect to UAT/PROD server:

```bash
ssh ubuntu@<UAT/PROD-public-ip>
```

Update the package list and install StrongSwan and required plugins:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install strongswan strongswan-starter strongswan-pki libcharon-extra-plugins -y
```

Verify the installation:

```bash
ipsec version
```

Output should show `Linux StrongSwan U5.9.12/K6.8.0....`.

- If you do not get the above output, run:
  ```bash
  sudo apt install strongswan-starter -y
  ```
  Then verify: `ipsec version`

---

## Step 2: Enable IP Forwarding

IP forwarding is required for the VPN to route packets.

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf
```

---

## Step 3: Configure `/etc/ipsec.conf`

Edit the configuration file:

```bash
sudo nano /etc/ipsec.conf
```

Replace the content with the following template. **Replace placeholders** with UAT/PROD actual values:

```
config setup
    charondebug="ike 2, knl 2, cfg 2"
    uniqueids=no

conn %default
    ikelifetime=28800s
    keylife=3600s
    rekeymargin=3m
    keyingtries=1
    keyexchange=ikev2
    authby=psk
    mobike=no

conn nbc-to-lolla
    left=<PRIVATE_IP>                 # e.g., 10.0.4.83
    leftid=<PUBLIC_IP>                # e.g., 63.178.83.38
    leftsubnet=<PUBLIC_IP>/32         # e.g., 63.178.83.38/32
    leftfirewall=yes
    right=102.212.82.5
    rightid=10.100.0.17               # NBC internal tunnel ID – do not change
    rightsubnet=196.45.159.14/32,196.45.159.17/32
    auto=start
    type=tunnel
    ike=aes256-sha256-ecp384!
    esp=aes256-sha256-ecp384!
    dpdaction=restart
    dpddelay=30s
    dpdtimeout=120s
```

> **Important:**  
> - `left` must be the **private IP** of the instance (the one assigned to `ens5`).  
> - `leftid` and `leftsubnet` must be the **public IP** (the one the bank will see).  
> - The `!` after algorithms enforces strict matching – keep it to ensure compatibility.

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

### Reference: Final Working NBC Config (PROD)

For your reference, this is the exact config that is deployed and working on the PROD server (`63.178.83.38`):

```
config setup
    charondebug="ike 2, knl 2, cfg 2"
    uniqueids=no

conn %default
    ikelifetime=28800s
    keylife=3600s
    rekeymargin=3m
    keyingtries=1
    keyexchange=ikev2
    authby=psk
    mobike=no

conn nbc-to-lolla
    left=10.0.4.83
    leftid=63.178.83.38
    leftsubnet=63.178.83.38/32
    leftfirewall=yes
    right=102.212.82.5
    rightid=10.100.0.17
    rightsubnet=196.45.159.14/32,196.45.159.17/32
    auto=start
    type=tunnel
    ike=aes256-sha256-ecp384!
    esp=aes256-sha256-ecp384!
    dpdaction=restart
    dpddelay=30s
    dpdtimeout=120s
```

---

## Step 4: Generate and Set the Pre-Shared Key (PSK)

### 4.1 Generate a Strong and Secure PSK

Use `openssl`:

```bash
openssl rand -base64 32
```

**Example output:**

```text
0hpGlESarMA2rGILIabqmdM+I6Ov3GHO61AbX4qwS7U=
```

This generates a 32‑byte (256‑bit) random key encoded in Base64. It is strong enough for production use.

### 4.2 Save the PSK Securely

Once you have generated the PSK, **save it in a secure location** (e.g., a password manager or encrypted vault). You will need it for:

- Configuring `/etc/ipsec.secrets` on your server.
- Sharing securely with NBC.

**Example of saved PSK (record for your reference):**

| Environment | Server IP | PSK |
| --- | --- | --- |
| UAT | 172.104.243.47 | `0hpGlESarMA2rGILIabqmdM+I6Ov3GHO61AbX4qwS7U=` |
| PROD | 63.178.83.38 | `<PSK_NBC_PROD>` |
| Test Server | 18.198.204.224 | `lolla_nbc=test123` |

### 4.3 Set the PSK in `/etc/ipsec.secrets`

Edit the secrets file:

```bash
sudo nano /etc/ipsec.secrets
```

Add the following line (replace `<PUBLIC_IP>` with UAT/PROD server’s public IP and `<YOUR_PSK>` with the generated key):

```text
<PUBLIC_IP> 102.212.82.5 : PSK "<YOUR_PSK>"
```

**Example for the test server:**

```text
18.198.204.224 102.212.82.5 : PSK "0hpGlESarMA2rGILIabqmdM+I6Ov3GHO61AbX4qwS7U="
```

**Example for UAT:**

```text
172.104.243.47 102.212.82.5 : PSK "0hpGlESarMA2rGILIabqmdM+I6Ov3GHO61AbX4qwS7U="
```

### 4.4 Set Proper Permissions

Ensure the secrets file is readable only by root:

```bash
sudo chmod 600 /etc/ipsec.secrets
```

### 4.5 Share the PSK Securely with NBC

**DO NOT send the PSK in plain email or inside the configuration file.** Use one of these secure methods:

1. **Encrypted email** (PGP/GPG) – if both parties support it.
2. **Signal / WhatsApp** – end‑to‑end encrypted messages.
3. **Phone call** – dictate the PSK verbally.
4. **Secure file transfer** (e.g., encrypted ZIP with a separate password).

> **Security:** The PSK must match exactly what is configured on NBC’s FortiGate. Exchange it securely via encrypted communication.

---

## Step 5: Firewall Configuration

We will enable `ufw` **without locking ourselves out** by first allowing our current SSH client IP.

> **Note:** If you rely solely on **AWS Security Groups** (Step 6), you can skip this step. However, running both provides defence‑in‑depth.

### 5.1 Find UAT/PROD Current SSH Client IP

Inside your SSH session, run:

```bash
echo $SSH_CLIENT | awk '{print $1}'
```

Note down this IP – you will allow it explicitly.

### 5.2 Add Firewall Rules (Before Enabling)

```bash
# Example: If the output from step 5.1 was 41.139.171.245

# Allow SSH from your specific client IP
sudo ufw allow from <YOUR_CLIENT_IP> to any port 22 proto tcp
# e.g.
sudo ufw allow from 41.139.171.245 to any port 22 proto tcp

# Allow VPN ports from NBC's public IP
sudo ufw allow from 102.212.82.5 to any port 500 proto udp
sudo ufw allow from 102.212.82.5 to any port 4500 proto udp

# Allow inbound test port 7782 from NBC subnets
sudo ufw allow from 196.45.159.14 to any port 7782 proto tcp
sudo ufw allow from 196.45.159.17 to any port 7782 proto tcp
```

> **Tip:** If your client IP changes often, you can allow SSH from anywhere (`sudo ufw allow 22/tcp`), but this is less secure.

### 5.3 Enable `ufw`

```bash
sudo ufw enable
```

When prompted `Command may disrupt existing ssh connections. Proceed? (y|n)`, type **`y`** and press Enter. Your current session stays active because you allowed your IP.

### 5.4 Verify Rules

```bash
sudo ufw status numbered
```

Expected output (showing your client IP and the VPN rules).

### 5.5 Test SSH in a New Terminal

**Open a new terminal** and try to reconnect:

```bash
ssh ubuntu@<UAT/PROD-public-ip>
```

If you succeed, proceed. If you get locked out, use the AWS EC2 Instance Connect or serial console to disable `ufw` (`sudo ufw disable`).

---

## Step 6: AWS Security Group (Cloud Firewall)

In the AWS Console, add inbound rules to the security group attached to the UAT/PROD instance:

| Type        | Protocol | Port Range | Source            |
|-------------|----------|------------|-------------------|
| Custom UDP  | UDP      | 500        | 102.212.82.5/32   |
| Custom UDP  | UDP      | 4500       | 102.212.82.5/32   |
| Custom TCP  | TCP      | 7782       | 196.45.159.14/32  |
| Custom TCP  | TCP      | 7782       | 196.45.159.17/32  |
| SSH         | TCP      | 22         | <YOUR-CLIENT-IP>/32 or 0.0.0.0/0 |

---

## Step 7: Add Public IP to Loopback (Permanent)

Because AWS NAT does not assign the public IP to any interface, the kernel will **drop** incoming packets for that IP unless we create a loopback alias.

### 7.1 Manual Addition (Temporary Test)

```bash
sudo ip addr add <UAT/PROD-PUBLIC_IP>/32 dev lo
```

Test with NBC – if they can connect, proceed to make it permanent.

### 7.2 Permanent via Systemd Service

Create a systemd service that adds the IP at boot:

```bash
sudo nano /etc/systemd/system/add-vpn-ip.service
```

Paste the following (use `replace` to avoid errors if the IP already exists):

```
[Unit]
Description=Add VPN public IP to loopback
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ip addr replace <PUBLIC_IP>/32 dev lo
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Example (test server):

```
[Unit]
Description=Add VPN public IP to loopback
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ip addr replace 18.198.204.224/32 dev lo
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl enable add-vpn-ip.service
sudo systemctl start add-vpn-ip.service
```

Check status:

```bash
sudo systemctl status add-vpn-ip.service
```

Verify the IP is present:

```bash
ip addr show lo
```

You should see `inet <PUBLIC_IP>/32 scope global lo`.

---

## Step 8: Start StrongSwan and Verify

```bash
sudo systemctl restart strongswan-starter
sudo systemctl enable strongswan-starter
```

Check the tunnel status:

```bash
sudo ipsec statusall
```

Look for:

- `Security Associations (1 up, 0 connecting)`
- `ESTABLISHED` for the IKE SA.
- `INSTALLED` Child SAs with UAT/PROD `leftsubnet` and the NBC subnets.

Also check the byte counters (`bytes_i` and `bytes_o`) – they may be zero until traffic flows.

---

## Step 9: Coordinate with NBC

Provide NBC with the following information if not captured in the setup form:

- UAT/PROD **public IP** (`<PUBLIC_IP>`).
- The **PSK** (agree on which PSK to use).
- Request them to **update their Phase‑2 Remote Address** to `<UAT/PROD-PUBLIC_IP>/32` (they may also need to remove any old IPs if they are no longer used).
- Confirm that IKE parameters match (AES256, SHA256, DH20).

Once they apply the changes, the tunnel should come up automatically (since `auto=start`). You can restart StrongSwan after they confirm:

```bash
sudo systemctl restart strongswan-starter
```

---

## Step 10: Test Inbound Connectivity

To verify that NBC can reach the UAT/PROD server through the tunnel, start a test listener:

```bash
sudo python3 -m http.server 7782
```

Ask NBC to attempt a connection (e.g., `telnet <PUBLIC_IP> 7782` or `nc -zv <PUBLIC_IP> 7782`).

- On UAT/PROD side, you can monitor packets:
  ```bash
  sudo tcpdump -i any port 7782 -n
  ```
- Check `ipsec statusall` – `bytes_o` should increase when you send replies.

If the connection succeeds, the VPN is fully operational.

---

## Replication for UAT / PROD

To set up the same VPN on a different AWS EC2 instance (e.g., UAT or PROD), **repeat all steps** but replace:

- `<PRIVATE_IP>` – new instance’s private IP.
- `<PUBLIC_IP>` – new instance’s public IP.
- The PSK may stay the same (if the bank allows) or use a new one (coordinate with them).
- Inform NBC of the new public IP so they can add an additional Phase‑2 selector.

---

# Part 2: Adding a Second Partner Bank — Stanbic Bank

This section shows how to **add Stanbic Bank as a second peer** alongside the already‑working NBC tunnel. **Do not modify the NBC `conn` stanza** — you are only **appending** a new `conn` block and a new PSK line.

Because StrongSwan supports multiple `conn` stanzas in `/etc/ipsec.conf`, both tunnels run simultaneously on the same AWS instance, using the same public IP (`63.178.83.38`) but different peers, algorithms, and Phase‑2 selectors.

## Overview of Stanbic’s Parameters

Extracted from the Stanbic form (SITE TO SITE VPN FORM – Stanbic DR and UAT):

| Parameter | Stanbic Value |
|-----------|---------------|
| Peer Public IP (`right`) | `196.8.216.18` |
| Peer Tunnel ID (`rightid`) | `196.8.216.18` (same as peer IP) |
| Peer Encryption Domain (their side) | `196.8.216.94/32` (UAT), `196.8.216.90/32` (DR) |
| Your Encryption Domain (`leftsubnet`) | `63.178.83.38/32` |
| Service Port | `7782/TCP` |
| Authentication Method | **Pre‑Shared Key** |
| IKE Version | IKEv2 |
| Phase 1 DH Group | **Group 19 (ECP256)** |
| Phase 1 Encryption | AES 256 |
| Phase 1 Hash | SHA2 (`sha256`) |
| Phase 1 Mode | Main mode |
| Phase 1 Lifetime | **86400s (1440 min)** |
| Phase 2 Encapsulation | ESP |
| Phase 2 Encryption | AES 256 |
| Phase 2 Authentication | SHA2 (`sha256`) |
| Phase 2 PFS | **NO PFS** |
| Phase 2 Lifetime | 3600s |

> **Key differences from NBC:** Different DH group (ECP256 vs ECP384), **no PFS**, and longer Phase‑1 lifetime. Make sure these are set correctly, or the tunnel will not come up.

---

## Step 11: Update `/etc/ipsec.conf` (Append Stanbic Conn)

Edit the file:

```bash
sudo nano /etc/ipsec.conf
```

**Keep the existing `config setup`, `conn %default`, and `conn nbc-to-lolla` sections unchanged.** Append the following `conn` block at the end of the file:

```
conn stanbic-to-lolla
    left=10.0.4.83                    # Same private IP as NBC
    leftid=63.178.83.38               # Same public IP
    leftsubnet=63.178.83.38/32        # Same local selector
    leftfirewall=yes

    right=196.8.216.18                # Stanbic Checkpoint public IP
    rightid=196.8.216.18              # Stanbic tunnel ID (same as peer IP)
    rightsubnet=196.8.216.94/32,196.8.216.90/32

    auto=start
    type=tunnel

    ike=aes256-sha256-ecp256!         # Group 19 = ECP256
    esp=aes256-sha256!                # No DH group in esp = PFS disabled
    pfs=no

    ikelifetime=86400s                # 1440 minutes
    keylife=3600s

    dpdaction=restart
    dpddelay=30s
    dpdtimeout=120s
```

### Why These Values

| Setting | Reason |
|---------|--------|
| `left=10.0.4.83` | Bind to the same private interface. Both conns share it. |
| `leftid=63.178.83.38` | Present the same public identity to Stanbic. |
| `leftsubnet=63.178.83.38/32` | The bank requires a public IP for interesting traffic; private ranges are not accepted. |
| `rightid=196.8.216.18` | Stanbic’s Checkpoint uses its public IP as identity (unlike NBC which uses an internal ID `10.100.0.17`). |
| `ike=...ecp256!` | Group 19 = ECP256, as specified in the Stanbic form. |
| `esp=aes256-sha256!` **and** `pfs=no` | Stanbic requires **NO PFS**. Omitting a DH group from `esp` and explicitly setting `pfs=no` is the correct way to disable PFS. |
| `ikelifetime=86400s` | 1440 minutes, per the form. |

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

### Reference: Full `/etc/ipsec.conf` After Adding Stanbic

Your final file should look like this (assuming PROD `63.178.83.38` / `10.0.4.83`):

```
config setup
    charondebug="ike 2, knl 2, cfg 2"
    uniqueids=no

conn %default
    ikelifetime=28800s
    keylife=3600s
    rekeymargin=3m
    keyingtries=1
    keyexchange=ikev2
    authby=psk
    mobike=no

conn nbc-to-lolla
    left=10.0.4.83
    leftid=63.178.83.38
    leftsubnet=63.178.83.38/32
    leftfirewall=yes
    right=102.212.82.5
    rightid=10.100.0.17
    rightsubnet=196.45.159.14/32,196.45.159.17/32
    auto=start
    type=tunnel
    ike=aes256-sha256-ecp384!
    esp=aes256-sha256-ecp384!
    dpdaction=restart
    dpddelay=30s
    dpdtimeout=120s

conn stanbic-to-lolla
    left=10.0.4.83
    leftid=63.178.83.38
    leftsubnet=63.178.83.38/32
    leftfirewall=yes
    right=196.8.216.18
    rightid=196.8.216.18
    rightsubnet=196.8.216.94/32,196.8.216.90/32
    auto=start
    type=tunnel
    ike=aes256-sha256-ecp256!
    esp=aes256-sha256!
    pfs=no
    ikelifetime=86400s
    keylife=3600s
    dpdaction=restart
    dpddelay=30s
    dpdtimeout=120s
```

---

## Step 12: Add Stanbic’s PSK to `/etc/ipsec.secrets`

Edit the secrets file:

```bash
sudo nano /etc/ipsec.secrets
```

**Keep the existing NBC line** and append a **new line for Stanbic**:

```
63.178.83.38 102.212.82.5 : PSK "<NBC_PSK>"
63.178.83.38 196.8.216.18 : PSK "<STANBIC_PSK>"
```

> Generate a **separate PSK** for Stanbic (`openssl rand -base64 32`) — do **not** reuse the NBC PSK.
>
> Ensure the file is still root‑only:
> ```bash
> sudo chmod 600 /etc/ipsec.secrets
> ```

---

## Step 13: Firewall / Security Group Updates for Stanbic

### 13.1 AWS Security Group

Add the following **inbound** rules in the AWS Console (in addition to the existing NBC rules):

| Type       | Protocol | Port Range | Source            | Purpose |
|------------|----------|------------|-------------------|---------|
| Custom UDP | UDP      | 500        | 196.8.216.18/32   | IKE (Phase 1) |
| Custom UDP | UDP      | 4500       | 196.8.216.18/32   | IPsec NAT‑T |
| Custom TCP | TCP      | 7782       | 196.8.216.94/32   | UAT service port |
| Custom TCP | TCP      | 7782       | 196.8.216.90/32   | DR service port |

### 13.2 `ufw` (if enabled)

If you are running `ufw`, add:

```bash
sudo ufw allow from 196.8.216.18 to any port 500 proto udp
sudo ufw allow from 196.8.216.18 to any port 4500 proto udp
sudo ufw allow from 196.8.216.94 to any port 7782 proto tcp
sudo ufw allow from 196.8.216.90 to any port 7782 proto tcp
sudo ufw reload
```

---

## Step 14: Restart StrongSwan and Verify Both Tunnels

```bash
sudo systemctl restart strongswan-starter
```

Check status:

```bash
sudo ipsec statusall
```

You should see **two connections**, and after the peers respond, **two sets of Security Associations**:

```
Connections:
nbc-to-lolla:      10.0.4.83...102.212.82.5   IKEv2
stanbic-to-lolla:  10.0.4.83...196.8.216.18   IKEv2

Security Associations (2 up, 0 connecting):
nbc-to-lolla[1]:      ESTABLISHED ...
stanbic-to-lolla[2]:  ESTABLISHED ...
```

Look for `INSTALLED` Child SAs under each connection:

- **NBC:** `63.178.83.38/32 === 196.45.159.14/32` and `...196.45.159.17/32`
- **Stanbic:** `63.178.83.38/32 === 196.8.216.94/32` and `...196.8.216.90/32`

If either is missing, check the logs:

```bash
sudo journalctl -u strongswan-starter -f
```

---

## Step 15: Coordinate with Stanbic

Fill the Stanbic form (column C – Lolla) as follows and return it to them:

| Field | Lolla Value |
|-------|-------------|
| **Primary Name** | Michael Muniu |
| **Primary Email** | mmuniu@abnosoftwares.com |
| **Primary Mobile** | +254 788 156 444 |
| **VPN Gateway IP Address** | `63.178.83.38` |
| **VPN Device Description** | StrongSwan on AWS EC2 (Ubuntu) |
| **VPN Device Version** | StrongSwan 5.9.x |
| **VPN Device Location** | AWS Cloud (eu-central-1) |
| **Encryption Domain** | `63.178.83.38/32` |
| **Service Port** | `7782 (TCP)` |
| **Authentication Method** | Pre‑Shared Key |
| **Encryption Scheme** | IKE v2 |
| **Diffie‑Hellman Group** | Group 19 |
| **Encryption Algorithm** | AES 256 |
| **Hashing Algorithm** | SHA256 |
| **Main or Aggressive Mode** | Main mode |
| **Phase 1 Lifetime** | 1440 minutes (86400 s) |
| **Phase 2 Encapsulation** | ESP |
| **Phase 2 Encryption** | AES 256 |
| **Phase 2 Authentication** | SHA256 |
| **Perfect Forward Secrecy** | NO PFS |
| **Phase 2 Lifetime** | 3600s |

Then send Stanbic an email with:

- **Your Public IP:** `63.178.83.38` (Remote Address on their firewall)
- **PSK:** send via Signal / encrypted email — **never** in plain email
- **Service Port:** `7782/TCP`
- Ask them to confirm when the Checkpoint policy is applied.

Once they confirm, restart StrongSwan:

```bash
sudo systemctl restart strongswan-starter
```

---

## Step 16: Test Inbound Connectivity from Stanbic

Start a listener on the service port:

```bash
sudo python3 -m http.server 7782
```

Ask Stanbic to connect (from their UAT `196.8.216.94` or DR `196.8.216.90`):

```
telnet 63.178.83.38 7782
```

Monitor packets on your side:

```bash
sudo tcpdump -i any port 7782 -n
```

You should see incoming `SYN` from `196.8.216.94`/`.90` and outgoing `SYN‑ACK` from `63.178.83.38`.  
Also verify `sudo ipsec statusall` — the **Stanbic** child SA’s `bytes_i` and `bytes_o` should increment.

---

## Stanbic‑Specific Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Phase 1 fails with `no acceptable proposal` | Wrong DH group (used ECP384 instead of ECP256) | Ensure `ike=aes256-sha256-ecp256!` (Group 19). |
| Phase 1 fails with `authentication failed` | PSK mismatch or wrong `rightid` | Confirm PSK matches; ensure `rightid=196.8.216.18` (public IP, not an internal ID). |
| Phase 2 fails with `no acceptable proposal` | PFS still enabled | Ensure `pfs=no` **and** `esp=aes256-sha256!` (no DH group). |
| Phase 2 fails with `TS_UNACCEPT` | Traffic selectors differ | Verify `leftsubnet=63.178.83.38/32` and `rightsubnet=196.8.216.94/32,196.8.216.90/32` — confirm Stanbic has added `63.178.83.38/32` as their Remote Address. |
| Tunnel up but no traffic | Public IP missing on loopback | Confirm `ip addr show lo` lists `63.178.83.38/32` (Step 7). |
| Only one tunnel established | `uniqueids` misconfigured | Keep `uniqueids=no` in `config setup` (already set). |
| Re‑key failures | Mismatched lifetimes | Stanbic requires `ikelifetime=86400s`, `keylife=3600s`. |

---

## Interaction Between the Two Tunnels

- Both `conn` blocks use the **same `left` / `leftid` / `leftsubnet`** but **different `right` peers**. StrongSwan keeps them separate because `right` is unique per conn.
- `uniqueids=no` is **critical** here — it allows the same local identity to be used with multiple peers simultaneously.
- Each peer will see the same public IP `63.178.83.38` on its side; the two tunnels do not interfere with each other’s encryption policies because their selectors and peers differ.
- **No changes** to the NBC conn are required when adding Stanbic, and vice versa.

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `sudo ipsec statusall` | Show all conns and installed SAs (both banks). |
| `sudo ipsec status nbc-to-lolla` | Show NBC tunnel only. |
| `sudo ipsec status stanbic-to-lolla` | Show Stanbic tunnel only. |
| `sudo ipsec restart` | Restart StrongSwan (all tunnels). |
| `sudo ipsec up stanbic-to-lolla` | Manually bring up the Stanbic tunnel. |
| `sudo ipsec down stanbic-to-lolla` | Manually tear down the Stanbic tunnel. |
| `sudo journalctl -u strongswan-starter -f` | Live logs (both tunnels). |
| `sudo tcpdump -i any port 500 or port 4500 -n` | Capture IKE/IPsec packets for all peers. |
| `sudo tcpdump -i any host 196.8.216.18 -n` | Capture traffic specific to Stanbic. |

---

**End of Documentation**