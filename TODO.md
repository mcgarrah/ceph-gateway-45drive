# TODO

Progress tracker against `INSTALL.md`.

- [ ] Step 1: Prepare the Gateway VM environment (Ubuntu 22.04, `ceph-common`, `attr`, `samba-common-bin`)
- [ ] Step 2: Securely mount Proxmox CephFS
  - [ ] Create scoped `client.gateway` cephx user (`ceph fs authorize`)
  - [ ] Save secret file on the gateway VM
  - [ ] Add `/etc/fstab` entry with real MON IPs
  - [ ] Verify mount (`df -h /mnt/cephfs`)
- [ ] Step 3: Install the 45Drives Cockpit stack
  - [ ] Add 45Drives repo, install `cockpit`, `samba`, `nfs-kernel-server`
  - [ ] Install `cockpit-file-sharing`, `cockpit-navigator`, `cockpit-identities`
  - [ ] Enable `cockpit.socket`
- [ ] Step 4: Provision SMB + NFSv4 shares via the Cockpit web UI
  - [ ] Create target subdirectory under `/mnt/cephfs`
  - [ ] Add SMB share
  - [ ] Add NFS export
- [ ] Step 5: Provision S3 via RADOS Gateway on the Gateway VM
  - [ ] Create scoped `client.rgw.<name>` cephx user
  - [ ] Add `/etc/ceph/ceph.conf` (mon_host + `[client.rgw.<name>]` section)
  - [ ] Install `radosgw`
  - [ ] Enable `ceph-radosgw@rgw.<name>`
  - [ ] Verify RGW is serving on :7480
  - [ ] Create a test S3 user, save access/secret keys
  - [ ] (Optional) Generate self-signed cert, add `ssl_port=7443` to `rgw_frontends`
  - [ ] (Optional) Verify TLS on :7443, retire plaintext :7480 once clients migrate

## Notes / issues encountered

(running log — add dated entries as you work through this)
