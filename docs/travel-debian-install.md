# Debian install

Base install for the travel server. Everything after this is handled by Ansible.

## Install media

1. Download the Debian 13 amd64 netinst ISO.
2. Verify the checksum:
   ```
   echo "<sha256>  debian-13.x.x-amd64-netinst.iso" | shasum -a 256 -c
   ```
3. Copy the ISO to the Ventoy USB.

## Installer

Text-mode **Install**, with ethernet connected.

- Network interface: wired ethernet
- Hostname: `travel`, domain blank
- Root password: blank (disables root, gives the user sudo)
- User: `andrewm`
- Partitioning: Guided, entire disk with LVM, on the 2TB system drive only
  - All files in one partition
  - Use `100 GB` of the volume group (the rest is left free for Ansible)
- Mirror: `deb.debian.org`
- Software: SSH server and standard system utilities only
- System clock set to UTC: yes

## SSH access

From the Mac:

```
ssh-keygen -R <server-ip>          # only if the IP was used by a previous install
ssh-copy-id andrewm@<server-ip>
ssh andrewm@<server-ip>
```

## Wipe the extra drive

One-time, by hand. Confirm the disk first, since this can't be undone.

```
lsblk -o NAME,SIZE,MODEL,FSTYPE,MOUNTPOINT
sudo vgs
sudo vgchange -an <old-vg>                 # if the disk has an old LVM group
sudo wipefs -a /dev/<disk>p*               # partition signatures
sudo wipefs -a /dev/<disk>                 # partition table
sudo blockdev --rereadpt /dev/<disk>
sudo update-grub                           # remove stale boot entries
lsblk /dev/<disk>                          # should show no partitions
```
