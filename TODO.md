# TODO

Progress tracker against `INSTALL.md`.

- [ ] Step 1: Prepare the Gateway VM environment (Ubuntu 22.04, `ceph-common`, `attr`, `samba-common-bin`)
- [ ] Step 2: Securely mount Proxmox CephFS
  - [ ] Extract admin keyring from a Proxmox node
  - [ ] Save secret file on the gateway VM
  - [ ] Add `/etc/fstab` entry with real MON IPs
  - [ ] Verify mount (`df -h /mnt/cephfs`)
- [ ] Step 3: Install the 45Drives Cockpit stack
  - [ ] Add 45Drives repo, install `cockpit`, `samba`, `nfs-kernel-server`
  - [ ] Install `cockpit-file-sharing`, `cockpit-navigator`, `cockpit-identities`
  - [ ] Enable `cockpit.socket`
- [ ] Step 4: Provision SMB + NFSv4 shares via the Cockpit web UI
- [ ] Step 5: Provision S3 via RADOS Gateway on the Proxmox host
  - [ ] Install `ceph-radosgw`
  - [ ] Enable `ceph-radosgw@rgw.<name>`
  - [ ] Create a test S3 user, save access/secret keys

## Notes / issues encountered

(running log — add dated entries as you work through this)
