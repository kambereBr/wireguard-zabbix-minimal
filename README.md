# WireGuard-Zabbix-Minimal Monitoring Template

A minimal, and flexible **Zabbix template** for monitoring **WireGuard VPN** interfaces and peers on Linux.
It is based on the open-source [`wg-json-zbx`](https://github.com/alkalim/wireguard-zabbix) script, enhanced to support Zabbix (6.4+) environments with customizable peer checks, version tracking, and configuration monitoring.

---

## Features

* Monitors WireGuard service status (`wg` command)
* Tracks WireGuard version
* Detects configuration changes (`wg0.conf` checksum)
* Checks peer handshake recency (per public key)
* Works without external dependencies - uses native Zabbix agent + `wg-json-zbx`
* All peers and parameters are configurable via Zabbix UI macros

---

## Installation (Debian/Ubuntu Example)

### 1. **Install Requirements**

```bash
sudo apt update
sudo apt install -y wireguard jq zabbix-agent
```

### 2. **Add Zabbix Permissions for `wg`**

Create `/etc/sudoers.d/zabbix-wireguard`:

```bash
Defaults:zabbix !requiretty
zabbix ALL=(root) NOPASSWD: /usr/bin/wg
```

Then apply:

```bash
sudo visudo -cf /etc/sudoers.d/zabbix-wireguard
```

---

### 3. **Install `wg-json-zbx`**

Fetch and install the script:

```bash
sudo mkdir -p /usr/share/zabbix
sudo wget -O /usr/share/zabbix/wg-json-zbx https://github.com/kambereBr/wireguard-zabbix-minimal/blob/main/wg-json-zbx
sudo chmod 755 /usr/share/zabbix/wg-json-zbx
sudo chown zabbix:zabbix /usr/share/zabbix/wg-json-zbx
```

> If you’ve tweaked the script, make sure it’s executable and outputs valid JSON.

---

### 4. **Add Zabbix User Parameters**

Create `/etc/zabbix/zabbix_agentd.d/user_parameters_wireguard.conf`:

```bash
UserParameter=wireguard.peers,/usr/share/zabbix/wg-json-zbx
# Count all peers across all WireGuard interfaces
UserParameter=wireguard.peers.count,/usr/share/zabbix/wg-json-zbx | jq '[.[] | .peers[]] | length'

# WireGuard service status
UserParameter=wireguard.service.status,systemctl is-active wg-quick@wg0.service | grep -q active && echo 1 || echo 0

# Command check (returns 1 if wg works, 0 if fails)
UserParameter=wireguard.command.status,/usr/bin/sudo /usr/bin/wg show all >/dev/null 2>&1; if [ $? -eq 0 ]; then echo 1; else echo 0; fi

# Last handshake elapsed (seconds) for a specific peer
UserParameter=wireguard.peer.last_handshake_elapsed[*],/usr/share/zabbix/wg-json-zbx | jq -r --arg key "$1" '.[] | .peers[] | select(.publicKey==$key) | ((now - (.latestHandshake // now)) | floor)'

# Config checksum (can be used for multiple interfaces)
UserParameter=wireguard.config.cksum[*],vfs.file.cksum[$1]

# WireGuard version
UserParameter=wireguard.version,/usr/bin/wg --version | head -1
```

Restart the agent:

```bash
sudo systemctl restart zabbix-agent
```

---

### 5. **Verify Agent Output**

Test data collection:

```bash
zabbix_get -s <WIREGUARD_HOST_IP> -k wireguard.command.status
zabbix_get -s <WIREGUARD_HOST_IP> -k wireguard.version
zabbix_get -s <WIREGUARD_HOST_IP> -k wireguard.peer.last_handshake_elapsed[<PEER-PUBLIC-KEY>]
```

---

### 6. **Import the Template**

1. In Zabbix UI → **Data Collection → Templates → Import**
2. Import `wireguard-minimal.yaml`
3. Link it to your WireGuard host.

---

## Configurable Macros (in Zabbix UI)

| Macro             | Description                       | Example                  |                     
| ----------------- | --------------------------------- | ------------------------ |
| `{$WG_PEER_NAME}` | Friendly peer name                | `Laptop 1`               |                    
| `{$WG_PEER_KEY}`  | Public key for handshake tracking | `<PEER-PUBLIC-KEY>`      |                     

You can duplicate the related items and triggers in the Zabbix template directly from the Zabbix UI to monitor multiple WireGuard peers.

---

## Included Triggers

| Trigger                                 | Description                                | Severity |
| --------------------------------------- | ------------------------------------------ | -------- |
| **WireGuard service not inactive**      | `wg` command or service not responding     | High     |
| **WireGuard config changed**            | `/etc/wireguard/wg0.conf` checksum changed | Warning  |
| **{$WG_PEER_NAME} no handshake for 5m** | No handshake received for 300s             | High     |
| **WireGuard version changed**           | WireGuard version has changed              | INFO     |

---

## Credits

* [**wg-json-zbx**](https://github.com/alkalim/wireguard-zabbix/blob/master/wg-json-zbx) by Alkalim - lightweight WireGuard JSON exporter
* [**WireGuard Tools**](https://github.com/WireGuard/wireguard-tools/blob/master/contrib/json/wg-json) by Jason A. Donenfeld
* Extended and maintained by **Bruno Kambere**
