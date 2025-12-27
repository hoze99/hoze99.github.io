---
title: Immich
categories: [homelab,immich]
image: /assets/img/immich.png
tags: [documentation,immich,lxc,proxmox,gpu,passthrough]
---
# How to run Immich in an LXC on Proxmox

## Immich
Immich is a self-hosted photography application. It acts as a replacement for Google Photos but resides within your own network. Immich provides face and object recognition features and works with both photo and video files. 

<em>"Easily back up, organize, and manage your photos on your own server. Immich helps you
browse, search and organize your photos and videos with ease, without sacrificing your privacy"</em>

The project website is [here:](https://immich.app/)

## My use cases
### Existing Photo Archive.
* I have thousands of Photographs stored on my file server. These files are stored in folders without any specific schema or ordering system. 
* I haven't built a Lightroom catalog or any other categorisation system.
* Consequently it is difficult to find photos of a specific event, or person, and searching for photos relating to a specific activity is not possible.

The immich application can scan this existing archive and build a searchable index/database of the existing photo archive. Immich supports searching for specific people, time periods, or activity types. 

### Photos captured on mobile devices.
* If the immich application is installed on our mobile phones, then the immich server can become the backup storage system for media captured on those devices. 
* We will then no longer need to rely on cloud storage services.

### Immich, docker, and GPU hardware.
Immich runs in docker, and to support offloading of machine learning and transcoding to a GPU, I setup a dedicated Linux Container on my main Proxmox host. 

 #### LXC creation
I used the Proxmox VE helper script to create an LXC with docker support. [Link](https://community-scripts.github.io/ProxmoxVE/scripts?id=docker)

This script will prompt for various settings including GPU passthrough. My Proxmox server has an NVidia GPU, as well as a 12 gen Intel CPU. Both options can be passed through to the LXC. 

The resources for this LXC have been extended to 8GB RAM and 4 vCPUs. Disk storage was also extended to 80GB. This provides sufficient space for the Immich index/database - which expands significantly when an external archive is setup. 

Note that Disk Storage will need to be extended further to cater for files uploaded from the mobile devices.

### External Library
Immich can process images and media files stored on an external library. In my case I want to use Immich to index the thousands of images stored on my file server.

The file server is a virtual machine running TrueNAS, and is hosted on the same Proxmox server. 

To make the file share available to the LXC running Immich, I first mounted the file share on the host Proxmox system, and then used a bind mount to pass the mounted file share through to the Immich LXC.

To mount the file share on the Proxmox host you might add a line to /etc/fstab like this:
``` bash
//192.168.1.1/photoshare /mnt/photos cifs username=user,password=password,vers=2.0,nofail,uid=1000,gid=1000,ro 0 0 
```
This will mount the share read only at /mnt/photos on the Proxmox host. Note the nofail option will avoid the Proxmos system stalling during boot if the share is unavailable.


To pass through the mounted share to your LXC, you could use a command like:
``` bash
pct set 100 -mp0 /mnt/photos,mp=/root/docker/immich-app/photos
```

In my case this configuration is flawed. When the proxmox host boots, it will be unable to mount the file share until the TrueNAS server is ready.

This issue has been addressed by first setting a startup priority on the TrueNAS VM.
The VM option Start/Shutdown order is set:
``` bash
Start/Shutdown order                 order=1,up=60
```
This sets the TrueNAS VM to start before any other VMs or LXCs. There is also a delay enforced so that the remaining VMs and LXCs don't start until 60 seconds after the TrueNAS VM startup.

On the Proxmox host a crontab entry has been added to mount all devices 180 seconds after boot, and then start the Immich LXC container.
``` bash
@reboot sleep 180; mount -a; /usr/sbin/pct start 100
```

See the boot timeline in the graphic below:
![Proxmox boot timeline](/assets/img/proxmox-startup-new.png)
