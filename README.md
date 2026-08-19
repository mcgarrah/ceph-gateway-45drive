# ceph-gateway-45drive

A protocol gateway VM (Ubuntu 22.04 LTS on Proxmox VE 9) that mounts a CephFS pool and
re-exposes it to the local network over SMB, NFSv4, and S3, using the 45Drives open-source
Cockpit stack (Houston UI plugins) for SMB/NFS and native Ceph RGW for S3.

See [`INSTALL.md`](INSTALL.md) for the step-by-step setup, and [`TODO.md`](TODO.md) for
current progress.

Spun off as a hands-on companion to the `snapshield-45drive` research project — building this
stack firsthand is what surfaced the "steep barrier of entry" finding documented in that
repo's Homelab Adoption Barriers section (the 45Drives open-source stack is ~15-20 separate
repos/plugins with no single install path).
