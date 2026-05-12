
#  Running ZeroTier on Synology Docker Behind a Strict Firewall

Running ZeroTier via Docker on a Synology NAS often results in infinite crash loops, offline statuses, or missing network interfaces. This happens due to Synology's strict file permissions, missing hardware devices, ghost network interfaces (`eth0`), and enterprise firewalls blocking UDP traffic.

This guide bypasses **all** of these restrictions to establish a stable, persistent ZeroTier connection.

## Prerequisites
* Synology NAS with **Container Manager** (Docker) installed.
* SSH access enabled (`Control Panel` > `Terminal & SNMP` > `Enable SSH service`).
* Your 16-character ZeroTier Network ID.

---

## Step 1: Create the Persistent TUN Device Script
Synology does not load the required virtual network driver (`tun.ko`) by default. We need a script that runs on boot to create this device.

1. SSH into your NAS and elevate to root:
   ```bash
   sudo -i
   ```

2. Create the startup script by pasting this entire block:
```bash
cat << 'EOF' > /usr/local/etc/rc.d/tun.sh
#!/bin/sh -e
insmod /lib/modules/tun.ko

if [ ! -c /dev/net/tun ]; then
  mkdir -p /dev/net
  mknod /dev/net/tun c 10 200
fi

chmod 666 /dev/net/tun
EOF

```


3. Make the script executable and run it immediately:
```bash
chmod a+x /usr/local/etc/rc.d/tun.sh
/usr/local/etc/rc.d/tun.sh

```


*(Note: If it says "File exists", that is normal. It means the module is already loaded.)*

---

## Step 2: Prepare the Data Folder & Fix Permissions

Synology's root file system blocks Docker containers from generating secret identity keys, causing crash loops. We must create a dedicated folder and grant it full permissions.

1. Create the data directory (assuming you use `volume1`):
```bash
mkdir -p /volume1/docker/zerotier/data

```


2. Grant read/write permissions to everyone so the container isn't blocked:
```bash
chmod -R 777 /volume1/docker/zerotier/data

```



---

## Step 3: Inject the Routing & Firewall Bypass Config

Many NAS devices have a disconnected `eth0` interface that acts as a black hole for ZeroTier traffic. Additionally, enterprise/university firewalls often block UDP port 9993.

We will create a `local.conf` file to force ZeroTier to ignore `eth0` and fall back to TCP relay servers to punch through strict firewalls.

1. Run this command to write the configuration file directly into the data folder we just made:
```bash
cat << 'EOF' > /volume1/docker/zerotier/data/local.conf
{
  "physical": {
    "eth0": { "blacklist": true }
  },
  "settings": {
    "allowTcpFallbackRelay": true,
    "portMappingEnabled": true,
    "primaryPort": 9993,
    "softwareUpdate": "disable"
  }
}
EOF

```



---

## Step 4: Deploy the Container (Do NOT use the DSM GUI!)

**Crucial:** Do not use Synology's Container Manager GUI to create this project. The GUI silently strips out hardware device mappings (`--device=/dev/net/tun`), which will cause the container to fail to build the virtual network interface.

Run this command in your SSH terminal to deploy the container with God-mode privileges:

```bash
docker run -d \
   --name zt \
   --restart=always \
   --net=host \
   --privileged \
   --cap-add=NET_ADMIN \
   --cap-add=SYS_ADMIN \
   --device=/dev/net/tun \
   -v /volume1/docker/zerotier/data:/var/lib/zerotier-one \
   zerotier/zerotier-synology:latest

```

---

## Step 5: Verify, Join, and Authorize

1. Give the container about 10 seconds to boot, then check its status:
```bash
docker exec -it zt zerotier-cli status

```


*You should see `200 info <Node_ID> 1.14.2 ONLINE` (or `TUNNELED` if the TCP fallback activated to bypass a firewall).*
2. Join your ZeroTier network:
```bash
docker exec -it zt zerotier-cli join <Your_Network_ID>

```


3. **Authorize the NAS:** Log into your ZeroTier Central dashboard (my.zerotier.com), find your new Node ID under the "Members" section, and check the box to authorize it. *(If it doesn't appear immediately due to the firewall, use the "Manually Add Member" button with your Node ID).*
4. **Final Verification:** Once authorized and assigned an IP by the dashboard, run:
```bash
ifconfig

```


Scroll to the bottom. You will see a `zt...` interface with your new ZeroTier IP address officially bound to your Synology NAS!

