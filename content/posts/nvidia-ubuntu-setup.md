+++
title = 'Nvidia setup on Ubuntu 2024 with secure boot enabled'
date = 2025-06-02T12:42:36+01:00
draft = false
+++

A quick guide for setting your gfx card up.
Hopefully painlessly.

After installing the nvidia toolkit (via instructions on their website)
and running `nvidia-smi`, we can't find the device.

The root of the problem is that the .509 cert used to sign the nvidia driver
is not added to the security chain of the secure boot.

To do this we need to use `mokutil`.

First we need to find out where the certificate is located.
One way of doing this is to remove the nvidia kernel module and install it back.
In the command's logs we should see the path to the cert.
It can for example be `/var/lib/shim-signed/mok/MOK.der`.

Then we can add that cert to be trusted:

```bash
sudo mokutil --test-key MOK.der    
# your cert should not be currently enrolled

sudo mokutil --import <your cert>
# mokutil should request pwd 

sudo mokutil --test-key MOK.der    
# your cert should be enrolled now 

sudo mokutil --list-new    
# your cert should be displayed

reboot
```

After the reboot, the MokManager should kick in and ask if we want to add the cert.
We'll need to provide the same password we used when importing the cert.
Then after the system boots up, we need to reload the module again:

```bash
sudo dkms remove nvidia/575.51.03 --all
sudo dkms install nvidia/575.51.03 -k $(uname -r)
sudo update-initramfs -u
sync
reboot
```

After the next reboot, the card should be visible to `nvidia-smi`.

Remember to enable systemd services for nvidia, so it works well with power management:

```bash
sudo systemctl enable nvidia-hibernate.service
sudo systemctl enable nvidia-persistenced.service
sudo systemctl enable nvidia-resume.service
sudo systemctl enable nvidia-suspend.service
```
