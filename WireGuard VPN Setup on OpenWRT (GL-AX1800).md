## 🔐 WireGuard VPN Setup on OpenWRT (GL-AX1800)

## 📌 Project Overview

This project documents how I set up a **WireGuard VPN server** on my GL.iNet GL-AX1800 (OpenWRT-based router) to securely connect my laptop to my home network. The goal was to enable remote access to my LAN and later use it for full internet tunneling.

---

## 🛠 Step 1. Installing WireGuard

On the router (OpenWRT):

`opkg update opkg install wireguard-tools luci-app-wireguard kmod-wireguard`

This installed the WireGuard kernel module, CLI tools, and LuCI web interface.

---

## 🛠 Step 2. Key Generation

On the router:

`wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key`

On the laptop:

`wg genkey | tee client_private.key | wg pubkey > client_public.key`

- **Private key** = stays on the device.
    
- **Public key** = shared with peers.

---

## 🛠 Step 3. Router Configuration

Edited `/etc/config/network` to create the `wg0` interface:


```
config interface 'wg0'
	option proto 'wireguard'     
	option private_key 'SERVER_PRIVATE_KEY'     
	list addresses '10.14.0.1/24'  
config wireguard_wg0     
	option public_key 'CLIENT_PUBLIC_KEY'     
	option description 'Laptop'    
	 list allowed_ips '10.14.0.2/32'
```

---

## 🛠 Step 4. Firewall Setup

Added WireGuard zone in `/etc/config/firewall`:

```
config zone     
	option name 'wg'     
	option input 'ACCEPT'     
	option forward 'ACCEPT'     
	option output 'ACCEPT'     
	option network 'wg0'  
config forwarding     
	option src 'wg'     
	option dest 'lan'  
	config forwarding     
	option src 'wg'     
	option dest 'wan'
```

Restarted firewall:

`/etc/init.d/firewall restart`

---

## Step 5. Client Configuration

Created `client.conf` on the laptop:

```
[Interface] PrivateKey = CLIENT_PRIVATE_KEY 
Address = 10.14.0.2/24 
DNS = 10.14.0.1  
[Peer] PublicKey = SERVER_PUBLIC_KEY 
Endpoint = <PUBLIC_IP_OR_DDNS>:41287 
AllowedIPs = 10.14.0.0/24 
PersistentKeepalive = 25`
```

---

### 🛠 Step 6. Problems Encountered & Fixes

#### ⚠️ Issue 1: `wg` not recognized on laptop

- **Problem:** Running `wg` on Windows gave “command not recognized.”
    
- **Cause:** WireGuard not installed on client.
    
- **Solution:** Installed the official **WireGuard for Windows** client, which provides GUI + CLI tools.
    

---

#### ⚠️ Issue 2: No handshake between router and laptop

- **Problem:** Tunnel activated but `wg show` showed no handshake.
    
- **Cause:** Laptop pointed to wrong port / ISP NAT.
    
- **Solution:**
    
    - Verified router listening port with `wg show`.
        
    - Opened/forwarded **UDP 41287** on router.
        
    - Corrected laptop `Endpoint` to match.
        
    - Handshake succeeded.
        

---

#### ⚠️ Issue 3: General failure when pinging VPN IP

- **Problem:** Tunnel up, but pinging `10.14.0.1` from laptop failed with “General failure.”
    
- **Cause:** `AllowedIPs` on client was missing `10.14.0.0/24`.
    
- **Solution:** Updated client config:
    
    `AllowedIPs = 10.14.0.0/24`
    

---

#### ⚠️ Issue 4: Could not ping LAN devices

- **Problem:** Could ping `10.14.0.1` (VPN router) but not `192.168.1.x` devices.
    
- **Cause:** No firewall forwarding between `wg` and `lan`.
    
- **Solution:** Added forwarding rules:

  ```
    config forwarding     
	    option src 'wg'    
	    option dest 'lan'  
	config forwarding     
		option src 'lan'     
		option dest 'wg'`
    ```
    Restarted firewall. LAN devices became reachable.
### 🛠 Step 7. Verification

- From laptop → `ping 10.14.0.1` ✅
    
- From laptop → `ping 192.168.1.100` (desktop PC) ✅
    
- `wg show` on router showed increasing TX/RX counters ✅
    

---

### 🔮 Next Steps

- Enable **full-tunnel mode** (`AllowedIPs = 0.0.0.0/0`) for secure internet browsing.
    
- Add **Dynamic DNS (DDNS)** so I don’t need to manually update public IP.
    
- Implement **internal CA certificates** for RDP & WireGuard for a trusted setup.
    

---

# ✅ Summary

I successfully set up **WireGuard VPN** on my GL-AX1800 router, connected my laptop as a client, and troubleshot multiple issues (missing client install, no handshake, wrong `AllowedIPs`, missing firewall rules). The tunnel now works reliably, giving me secure LAN access over VPN.