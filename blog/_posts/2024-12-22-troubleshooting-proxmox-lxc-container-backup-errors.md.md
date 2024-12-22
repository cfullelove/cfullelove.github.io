---
layout: post
title: "Troubleshooting Proxmox LXC Container Backup Errors"
published: true
post_date: 2022-01-04 12:00
---

Today, I encountered a perplexing issue while attempting to back up two LXC containers in Proxmox. These containers threw errors during the backup process, and after a bit of trial and error, I managed to resolve the issue. I hope that by sharing my experience, others may avoid the same pitfalls.

### The Problem: Backup Errors and Permission Denied Messages

My journey began when I noticed repeated failures in my Proxmox backup logs for container `206`. The error message was as follows:

```
INFO: Starting Backup of VM 206 (lxc)
INFO: Backup started at 2024-12-22 18:00:20
INFO: tar: ./home/ansible/.ansible: Cannot open: Permission denied
INFO: tar: ./home/ansible/.ssh: Cannot open: Permission denied
ERROR: Backup of VM 206 failed - command [...] failed: exit code 2
```

The logs indicated that files within the container's home directory could not be opened due to permission issues. Despite my best efforts searching through articles, none provided the clarity I needed. Fortunately, a Proxmox forum post [here](https://forum.proxmox.com/threads/vzdump-permission-denied.128951/) pointed me in the right direction.

### The Root Cause: Altered User ID Mapping

Upon further investigation, I discovered that the issue stemmed from altered user ID mappings after the containers' creation. Essentially, I adjusted the user mappings in my LXC containers such that user `1000` inside the container was mapped to a different user on the Proxmox host. This change wasn't initially problematic, but it inadvertently led to ownership conflicts because the files created before modifying the user mappings retained the original user ID ownership.

For container `206`, the user mapping configuration in `/etc/pve/lxc/206.conf` looked like this:

```
lxc.idmap: u 0 100000 1000
lxc.idmap: g 0 100000 1000
lxc.idmap: u 1000 1000 1
lxc.idmap: g 1000 1000 1
lxc.idmap: u 1001 101001 64535
lxc.idmap: g 1001 101001 64535
```

### The Initial State of Directory Permissions

The misalignment between the expected and actual ownership meant that the files had the wrong UID, preventing the backup tool from accessing them. Before fixing, the files in the container's home directory had the following permissions:

```
root@real4:\~# ls -la /tank/subvol-206-disk-0/home/ansible
total 31
drwxr-xr-x 4 101000 101000    7 May  5  2024 .
drwxr-xr-x 3 100000 100000    3 May  5  2024 ..
drwx------ 3 101000 101000    3 May  5  2024 .ansible
-rw-r--r-- 1 101000 101000  220 Mar 28  2022 .bash_logout
-rw-r--r-- 1 101000 101000 3526 Mar 28  2022 .bashrc
-rw-r--r-- 1 101000 101000  807 Mar 28  2022 .profile
drwx------ 2 101000 101000    3 May  5  2024 .ssh
```

The UIDs were `101000` instead of `1000`, which caused the permission issues during the backup process.

### The Solution: Correcting File Ownership

To resolve the permission issues, I had to correct the ownership of the files in question. Here's what I did:

```
# chown -R 1000:1000 /tank/subvol-206-disk-0/home/ansible
```

This command recursively changed the ownership of the files in the directory to the correct UID and GID (`1000:1000`). Post-adjustment, my containers backed up successfully, and the permissions were as they should be:

```
root@real4:\~# ls -lan /tank/subvol-206-disk-0/home/ansible
total 31
drwxr-xr-x 4   1000   1000    7 May  5  2024 .
drwxr-xr-x 3 100000 100000    3 May  5  2024 ..
drwx------ 3   1000   1000    3 May  5  2024 .ansible
-rw-r--r-- 1   1000   1000  220 Mar 28  2022 .bash_logout
-rw-r--r-- 1   1000   1000 3526 Mar 28  2022 .bashrc
-rw-r--r-- 1   1000   1000  807 Mar 28  2022 .profile
drwx------ 2   1000   1000    3 May  5  2024 .ssh
```

### Conclusion

If you encounter backup errors in Proxmox with similar permission denied messages, consider checking your user ID mappings, particularly if these have been altered post-container creation. Correcting file ownership based on the adjusted UID mappings can potentially save your day. I hope this writeup helps someone else dealing with a similar conundrum. Have questions or tips to share? Feel free to leave a comment below!