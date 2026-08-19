# INSTALL.md: Ceph Storage Gateway on Ubuntu 22.04 LTS

This guide details configuring an Ubuntu 22.04 LTS Virtual Machine (VM) running on Proxmox 9 to act as a protocol gateway. It securely mounts a CephFS storage pool and surfaces **SMB, NFSv4, and S3** interfaces to local network clients using the 45Drives open-source software stack.

## Architecture Prerequisites
* **Hypervisor:** Proxmox VE 9 cluster with an active, working Ceph configuration.
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

1. **Extract your credentials from Proxmox:**
   Log into any of your physical Proxmox nodes and view your Ceph administrative keyring file (`/etc/ceph/ceph.client.admin.keyring`). Copy the alphanumeric string found next to `key = `.

2. **Save the key file on your Gateway VM:**
   On your Ubuntu VM, create a path to hold this access credential securely:
   ```bash
   sudo mkdir -p /etc/ceph
   echo "YOUR_COPIED_KEY_STRING_HERE" | sudo tee /etc/ceph/ceph.client.admin.secret
   sudo chmod 600 /etc/ceph/ceph.client.admin.secret
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
   MON_IP_1:6789,MON_IP_2:6789,MON_IP_3:6789:/ /mnt/cephfs ceph name=admin,secretfile=/etc/ceph/ceph.client.admin.secret,noatime,_netdev 0 0
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

## Step 5: Provision S3 Object Gates (Proxmox Native Path)

To avoid breaking python translations on 45Drives plugins, handle cloud-native object endpoints directly through the underlying Ceph core architecture via your Proxmox environment.

1. **Deploy the RADOS Gateway:**
   Log directly into your Proxmox 9 host node terminal and run:
   ```bash
   sudo apt update && sudo apt install ceph-radosgw -y
   ```
2. **Bootstrap the Service Instance:**
   Create the gateway entry point. Replace `gateway-name` with your hostname modifier:
   ```bash
   sudo systemctl enable --now ceph-radosgw@rgw.gateway-name
   ```
3. **Provision Object Storage Users:**
   Generate active keys to connect external local applications:
   ```bash
   radosgw-admin user create --uid="local-user" --display-name="Local Storage User"
   ```
   Save the resulting JSON output blocks locally; they contain the explicit `access_key` and `secret_key` parameters required to bind third-party client backup tools over S3 using port `7480`.
