# StrongSwan IPsec VPN Setup for AWS EC2 (Site‑to‑Site with NBC Bank)

## Overview

This document provides a step‑by‑step guide to replicate the IPsec IKEv2 VPN tunnel between an **AWS EC2 Ubuntu instance** (running StrongSwan) and the **FortiGate firewall** at NBC Bank. The setup is identical for both **UAT** and **PROD** environments; you only need to adjust the IP addresses.

### Architecture Summary

- The EC2 instance has a **private IP** (e.g., `10.0.38.237`) and a **public IP** (e.g., `18.198.204.224`) via AWS NAT.
- StrongSwan binds to the private IP, but presents the public IP as its identity and traffic selector.
- NBC’s FortiGate uses the public IP as the remote address in Phase 2.
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

- If you do not get the above output, run
```bash
sudo apt install strongswan-starter -y
```
Then verify the installation ```ipsec version ```

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
    left=<PRIVATE_IP>                 # e.g., 10.0.38.237
    leftid=<PUBLIC_IP>                # e.g., 18.198.204.224
    leftsubnet=<PUBLIC_IP>/32         # e.g., 18.198.204.224/32
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

```ctrl+O ```, ```Enter```, then ```ctrl+X ``` to Save and exit 

---

## Step 4: Generate and Set the Pre-Shared Key (PSK)

### 4.1 Generate a Strong and Secure PSK

#### Use `openssl`

```bash
openssl rand -base64 32
```

**Example output:**

```text
0hpGlESarMA2rGILIabqmdM+I6Ov3GHO61AbX4qwS7U=
```

This generates a 32-byte (256-bit) random key encoded in Base64. It is strong enough for production use.

### 4.2 Save the PSK Securely

Once you have generated the PSK, **save it in a secure location** (e.g., a password manager or encrypted vault). You will need it for:

- Configuring `/etc/ipsec.secrets` on your server.
- Sharing securely with NBC.

**Example of saved PSK (record for your reference):**

| Environment | Server IP | PSK |
| --- | --- | --- |
| UAT | 172.104.243.47 | `0hpGlESarMA2rGILIabqmdM+I6Ov3GHO61AbX4qwS7U=` |
| PROD | 63.178.83.38 | `xK9mPqR5tNvW8zYbA2cF6hJ4sT3uM7pL` |
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
2. **Signal / WhatsApp** – end-to-end encrypted messages.
3. **Phone call** – dictate the PSK verbally.
4. **Secure file transfer** (e.g., encrypted ZIP with a separate password).
> **Security:** The PSK must match exactly what is configured on NBC’s FortiGate. Exchange it securely via encrypted communication.

---

## Step 5: Firewall Configuration

We will enable `ufw` **without locking ourselves out** by first allowing our current SSH client IP.

### 5.1 Find UAT/PROD Current SSH Client IP

Inside your SSH session, run:

```bash
echo $SSH_CLIENT | awk '{print $1}'
```

Note down this IP – you will allow it explicitly.

### 5.2 Add Firewall Rules (Before Enabling)

```bash
#Example: If the output from step 5.1 was 41.139.171.245

# Allow SSH from your specific client IP
sudo ufw allow from <YOUR_CLIENT_IP> to any port 22 proto tcp

e.g., sudo ufw allow from 41.139.171.245 to any port 22 proto tcp

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

In the AWS Console, add inbound rules to the security group attached to UAT/PROD instance:

| Type        | Protocol | Port Range | Source            |
|-------------|----------|------------|-------------------|
| Custom UDP  | UDP      | 500        | 102.212.82.5/32   |
| Custom UDP  | UDP      | 4500       | 102.212.82.5/32   |
| Custom TCP  | TCP      | 7782       | 196.45.159.14/32  |
| Custom TCP  | TCP      | 7782       | 196.45.159.17/32  |
| SSH  | TCP | 22        | <YOUR - CLIENT - IP> or 0.0.0.0/0 |

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
Example entries made on the test server:
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

Provide NBC with the following information if not captured in the set up form:

- UAT/PROD **public IP** (`<PUBLIC_IP>`).
- The **PSK** (Agree on which PSK to use).
- Request them to **update their Phase‑2 Remote Address** to `<UAT/PROD-PUBLIC_IP>/32` (they may also need to remove any old IPs if they are no longer used).
- Confirm that IKE parameters match (AES256, SHA256, DH20).

Once they apply the changes, the tunnel should come up automatically (since `auto=start`). You can restart StrongSwan after they confirm:

```bash
sudo systemctl restart strongswan-starter
```

---

## Step 10: Test Inbound Connectivity

To verify that NBC can reach UAT/PROD server through the tunnel, start a test listener:

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

## Troubleshooting Cheatsheet

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Phase 1 fails (`no acceptable proposal`) | Algorithm mismatch | Verify `ike=` and `esp=` lines match NBC’s proposal. Try removing `!` to be more permissive, or double‑check DH group. |
| Phase 1 fails (`authentication failed`) | PSK mismatch or wrong `rightid` | Ensure PSK matches exactly and `rightid=10.100.0.17`. |
| Phase 2 fails (`TS_UNACCEPT`) | Traffic selector mismatch | Ensure `leftsubnet` equals UAT/PROD public IP/32 and NBC has added that IP as Remote Address. |
| Tunnel up but no traffic | Public IP not on loopback | Add IP to `lo` (see Step 7). |
| No incoming SYN‑ACK replies | `ufw` blocking or kernel not accepting IP | Check `ufw` rules and the loopback alias. |
| Connection works, then stops | Re‑authentication issue | Check logs; ensure `dpdaction=restart` and firewall allows keep‑alive. |
| StrongSwan not starting | Syntax error in config | Run `sudo ipsec start --nofork` to see errors. |

### Useful Commands

| Command | Purpose |
|---------|---------|
| `sudo ipsec statusall` | Show full tunnel status and installed SAs. |
| `sudo journalctl -u strongswan-starter -f` | Real‑time logs. |
| `sudo tcpdump -i any port 500 or port 4500 -n` | Capture IKE/ESP packets. |
| `sudo tcpdump -i any port 7782 -n` | Monitor test traffic. |
| `ip addr show lo` | Verify loopback alias. |
| `sudo ufw status numbered` | List firewall rules. |
| `sudo systemctl status add-vpn-ip.service` | Check loopback alias service. |

---

## Final Notes

- This documentation is tailored for AWS EC2 with NAT. If you ever deploy on a server with a directly‑assigned public IP, you can set `left=<PUBLIC_IP>` instead of the private IP.
- Always test the firewall rules carefully (Step 5) to avoid losing SSH access.
- Keep the loopback alias service enabled so the IP survives reboots.

---

**End of Documentation**
