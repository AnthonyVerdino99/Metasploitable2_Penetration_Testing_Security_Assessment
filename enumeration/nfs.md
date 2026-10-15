# NFS Enumeration

## Objective

Determine whether the NFS service permits access to exported filesystem resources.

## Enumeration

NFS was identified on TCP/2049.

The target's NFS exports were identified using:

```bash
showmount -e 172.30.1.21

The server returned:

Export list for 172.30.1.21:
/ *

The root filesystem was then mounted read-only:

sudo mount -t nfs -o ro,nolock 172.30.1.21:/ /mnt/metasploitable-nfs
Findings

The NFS configuration exposed the target's root filesystem to network clients.

Read-only enumeration exposed system directories including:

/etc
/home
/root
/usr
/var

The /home directory exposed:

ftp
msfadmin
service
user

Additional system information was accessible through the exported filesystem, including the target hostname and operating system information.

Security Impact

An unrestricted NFS root filesystem export can expose sensitive system and user information to network clients.

Depending on the export configuration and filesystem permissions, this type of exposure may also provide opportunities for further compromise.

Evidence
reconnaissance/nmap-nfs-enumeration.txt
screenshots/nfs/nfs-root-filesystem.png
