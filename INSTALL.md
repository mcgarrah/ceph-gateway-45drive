# INSTALL.md: Ceph Storage Gateway on Ubuntu 22.04 LTS

This guide details configuring an Ubuntu 22.04 LTS Virtual Machine (VM) running on Proxmox VE 8.4 or 9 to act as a protocol gateway. It securely mounts a CephFS storage pool and surfaces **SMB, NFSv4, and S3** interfaces to local network clients using the 45Drives open-source software stack.

## Architecture Prerequisites
* **Hypervisor:** Proxmox VE 8.4 or 9 cluster with an active, working Ceph configuration. Both ship a compatible Ceph Squid release, and everything the gateway VM does — kernel CephFS mount, cephx auth, RGW as a plain Ceph client — talks to the cluster over the standard Ceph client protocol, not `pveceph`/host-side tooling, so it isn't sensitive to which of the two you're running.
* **Storage backend:** An active CephFS pool managed on the Proxmox layer (requires an active Metadata Server/MDS).
* **Gateway Instance:** A clean Ubuntu 22.04 LTS VM with its network adapter bound to a local Proxmox bridge (`vmbr0`) for explicit local IP assignment.

---

## Step 1: Prepare the Gateway VM Environment

Log directly into your Ubuntu 22.04 VM shell. Update the package database and install critical file-system utility dependencies:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ceph-common attr samba-common-bin
```

---

## Step 2: Securely Mount Proxmox CephFS

To share file storage over SMB and NFS, you must first link the VM's kernel filesystem to the cluster's CephFS directory.

1. **Create a scoped cephx user on a Proxmox/Ceph admin node:**
   Do not use the cluster admin keyring on the gateway VM. Instead, mint a dedicated `client.gateway` user restricted to the CephFS filesystem you're exporting:
   ```bash
   ceph fs authorize cephfs client.gateway / rw
   ```
   Replace `cephfs` with your filesystem name if different. This grants `client.gateway` read/write MDS and OSD caps scoped to that filesystem only (no mon/osd admin, no access to other pools).

2. **Retrieve the key and save it on your Gateway VM:**
   On the admin node:
   ```bash
   ceph auth print-key client.gateway
   ```
   On your Ubuntu VM, create a path to hold this access credential securely:
   ```bash
   sudo mkdir -p /etc/ceph
   echo "YOUR_COPIED_KEY_STRING_HERE" | sudo tee /etc/ceph/ceph.client.gateway.secret
   sudo chmod 600 /etc/ceph/ceph.client.gateway.secret
   ```

3. **Configure Persistent Boot Mounts:**
   Create the target workspace path on the VM directory tree:
   ```bash
   sudo mkdir -p /mnt/cephfs
   ```
   Open your system's filesystem configuration table:
   ```bash
   sudo nano /etc/fstab
   ```
   Append the following line to the bottom of the file. Replace `MON_IP_1`, `MON_IP_2`, and `MON_IP_3` with the actual LAN IP addresses of your Proxmox Ceph Monitor nodes:
   ```text
   MON_IP_1:6789,MON_IP_2:6789,MON_IP_3:6789:/ /mnt/cephfs ceph name=gateway,secretfile=/etc/ceph/ceph.client.gateway.secret,noatime,_netdev 0 0
   ```
   *(Note: The `_netdev` parameter forces the OS to delay mounting until after network interfaces are online).*

4. **Verify the Mount:**
   Force the OS to parse the new table rules immediately:
   ```bash
   sudo mount -a
   df -h /mnt/cephfs
   ```
   Verify that your full pool size maps accurately inside the output window.

---

## Step 3: Install the 45Drives Cockpit Stack

Use the verified [45Drives Repositories Script](https://knowledgebase.45drives.com/kb/kb451400-using-45drives-repositories/) to establish correct package signkeys for Ubuntu Jammy.

1. **Inject Repositories & Install Baseline UI:**
   ```bash
   curl -sSL https://repo.45drives.com/setup | sudo bash
   sudo apt update
   sudo apt install -y cockpit samba nfs-kernel-server
   ```

2. **Deploy 45Drives Specialized Modules:**
   Install the custom web plugins used to map network sharing rules and storage entities:
   ```bash
   sudo apt install -y cockpit-file-sharing cockpit-navigator cockpit-identities
   ```

3. **Launch the Gateway Daemon:**
   Force Cockpit to initiate and run persistently across power failures:
   ```bash
   sudo systemctl enable --now cockpit.socket
   ```

---

## Step 4: Provision Services via the Web Interface

Open a web browser on your network and point it to the VM's specific instance URL:
`https://<GATEWAY_VM_IP_ADDRESS>:9090`

Log in using your **Ubuntu system credentials** (ensure the user account belongs to the standard `sudo` group).

### Configuring SMB Shares (Windows/macOS Compatible)
0. If it doesn't already exist, create the target subdirectory first: `sudo mkdir -p /mnt/cephfs/your_target_folder`.
1. Select the **File Sharing** module inside the Cockpit side navigation window.
2. Select the **Samba** tab and choose **Add Share** (`+`).
3. Set the directory path specifically to point to your cluster subdirectory: `/mnt/cephfs/your_target_folder`.
4. Define your preferred Access Control rules and click **Confirm**.

### Configuring NFSv4 Shares (Linux/Unix Compatible)
1. Select the **NFS** tab under the File Sharing menu space.
2. Click **Add Export** (`+`).
3. Set the path to your cluster volume `/mnt/cephfs/`.
4. Explicitly limit client interaction boundaries by defining your local subnet address rules (e.g., `192.168.1.0/24`) under the permitted client access settings.

---

## Step 5: Provision S3 Object Gateway (RADOS Gateway on the Gateway VM)

To keep all client-facing protocols (SMB, NFS, S3) terminating on a single instance, RGW runs on the Ubuntu gateway VM itself rather than on the Proxmox host. This is additive to the Ceph cluster — it does not modify or replace anything managed by `pveceph` — but it does mean RGW's lifecycle (installs, upgrades, restarts) is handled manually on the VM instead of through the Proxmox Ceph GUI.

> **Caveat — HTTP only:** steps 1-7 below configure the `beast` frontend for plain HTTP on port `7480`, with no TLS. S3 access keys, secret keys, and object data all travel unencrypted. That's an acceptable tradeoff on a trusted local LAN, but do not expose port 7480 beyond it — anyone on the same network segment can sniff credentials in transit. Steps 8-10 add TLS with a self-signed certificate if you want it.

1. **Create a scoped cephx user for RGW:**
   On a Proxmox/Ceph admin node, mint a dedicated identity for the gateway instance (replace `gateway-name` with your hostname modifier). Omit `-o` so the full keyring block prints to your terminal for you to copy:
   ```bash
   ceph auth get-or-create client.rgw.gateway-name mon 'allow rw' osd 'allow rwx'
   ```
   This grants only what RGW needs to operate — no admin caps. Note the `osd` cap is necessarily broader than the CephFS client's in Step 2, since RGW auto-creates and manages its own `.rgw.*` pools at runtime.

2. **Provide cluster connection info on the Gateway VM:**
   The kernel CephFS mount from Step 2 didn't require a `ceph.conf`, but RGW does. Create one with your monitor addresses:
   ```bash
   sudo tee /etc/ceph/ceph.conf <<EOF
   [global]
   mon_host = MON_IP_1,MON_IP_2,MON_IP_3
   EOF
   ```
   Paste the keyring block from step 1 into a keyring file on the VM and lock down its permissions:
   ```bash
   sudo tee /etc/ceph/ceph.client.rgw.gateway-name.keyring <<EOF
   [client.rgw.gateway-name]
       key = YOUR_COPIED_RGW_KEY_STRING_HERE
   EOF
   sudo chmod 600 /etc/ceph/ceph.client.rgw.gateway-name.keyring
   ```

3. **Install the RADOS Gateway package on the Gateway VM:**
   ```bash
   sudo apt update && sudo apt install -y radosgw
   ```

4. **Add the RGW instance config to `ceph.conf`:**
   `host` must match the gateway VM's `hostname -s` output.
   ```bash
   sudo tee -a /etc/ceph/ceph.conf <<EOF

   [client.rgw.gateway-name]
   host = gateway-vm-hostname
   rgw_frontends = "beast port=7480"
   keyring = /etc/ceph/ceph.client.rgw.gateway-name.keyring
   EOF
   ```

5. **Enable and start the RGW daemon:**
   ```bash
   sudo systemctl enable --now ceph-radosgw@rgw.gateway-name
   ```

6. **Verify RGW is serving:**
   ```bash
   sudo systemctl status ceph-radosgw@rgw.gateway-name --no-pager
   curl -s http://localhost:7480
   ```
   A `ListAllMyBucketsResult` (anonymous access denied) XML response confirms the daemon is up and listening on 7480.

7. **Provision Object Storage Users:**
   Run directly on the gateway VM now that `ceph-common`/`radosgw-admin` are local:
   ```bash
   sudo radosgw-admin user create --uid="local-user" --display-name="Local Storage User"
   ```
   Save the resulting JSON output blocks locally; they contain the explicit `access_key` and `secret_key` parameters required to bind third-party client backup tools over S3 using port `7480` against the gateway VM's IP.

8. **(Optional) Generate a self-signed certificate for TLS:**
   Good enough to stop plaintext credential sniffing on a trusted LAN; it won't validate against a public CA, so LAN clients will need to either trust it explicitly or skip verification. Replace `gateway-vm-hostname` and `GATEWAY_VM_IP_ADDRESS` with your VM's real hostname and IP:
   ```bash
   sudo mkdir -p /etc/ceph/rgw-tls
   sudo openssl req -x509 -nodes -newkey rsa:4096 -days 3650 \
     -keyout /etc/ceph/rgw-tls/rgw.key \
     -out /etc/ceph/rgw-tls/rgw.crt \
     -subj "/CN=gateway-vm-hostname" \
     -addext "subjectAltName=DNS:gateway-vm-hostname,IP:GATEWAY_VM_IP_ADDRESS"
   ```
   Beast's `ssl_certificate` option expects one PEM file containing the certificate followed by the private key, so concatenate them:
   ```bash
   sudo sh -c 'cat /etc/ceph/rgw-tls/rgw.crt /etc/ceph/rgw-tls/rgw.key > /etc/ceph/rgw-tls/rgw-combined.pem'
   sudo chmod 600 /etc/ceph/rgw-tls/rgw-combined.pem
   ```

9. **Add an SSL listener to the RGW frontend and restart:**
   Add `ssl_port` and `ssl_certificate` to the `rgw_frontends` line set in step 4:
   ```bash
   sudo sed -i \
     's|rgw_frontends = "beast port=7480"|rgw_frontends = "beast port=7480 ssl_port=7443 ssl_certificate=/etc/ceph/rgw-tls/rgw-combined.pem"|' \
     /etc/ceph/ceph.conf
   sudo systemctl restart ceph-radosgw@rgw.gateway-name
   ```
   Leaving plain `port=7480` in place alongside `ssl_port=7443` lets you cut clients over one at a time; once everything speaks `https://<GATEWAY_VM_IP>:7443`, drop `port=7480` from the line and restart again to close off the unencrypted listener.

10. **Verify TLS:**
    ```bash
    curl -sk https://localhost:7443
    ```
    `-k` skips certificate validation, since it's self-signed. For clients that shouldn't skip validation, distribute `/etc/ceph/rgw-tls/rgw.crt` and have them trust it explicitly rather than using `-k`/insecure mode long-term.

---

## Future Work: Firewall / ufw (Not Yet Configured)

This guide deliberately does not enable `ufw` or add any firewall rules on the gateway VM — that's out of scope until the gateway is validated end-to-end on an open LAN first. When it's time to lock this down, the ports involved are:

| Port | Protocol | Service |
|---|---|---|
| `9090/tcp` | HTTPS | Cockpit web UI |
| `445/tcp`, `139/tcp` | SMB | Samba |
| `2049/tcp` | NFSv4 | nfs-kernel-server |
| `7480/tcp` | HTTP | RGW (plaintext) |
| `7443/tcp` | HTTPS | RGW, if TLS was enabled in step 8-10 above |

The eventual rules should scope each of these to the local subnet only (e.g. `ufw allow from 192.168.1.0/24 to any port 9090`) rather than opening them broadly, and should drop `7480` entirely once RGW clients have migrated to `7443`. Not implemented here.
