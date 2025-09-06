Create `/etc/modprobe.d/disable-nvidia.conf` including:
```
install nvidia /bin/false
```
Finally add nvidia and ipmi in the modprobe.d blacklist to disable this functionality, edit `/etc/modprobe.d/blacklist.conf`
```
blacklist nouveau
blacklist rivafb
blacklist nvidiafb
blacklist rivatv
blacklist nv
blacklist nvidia
blacklist nvidia-drm
blacklist nvidia-modeset
blacklist nvidia-uvm
```
Create 2 GPU management scripts for enabling and disabling discret NVIDIA graphical card [4]:

#### `enablegpu.sh`:
```
#!/bin/sh
# allow to load nvidia module
mv /etc/modprobe.d/disable-nvidia.conf /etc/modprobe.d/disable-nvidia.conf.disable

# Remove NVIDIA card (currently in power/control = auto)
echo -n 1 > /sys/bus/pci/devices/0000\:01\:00.0/remove
sleep 1
# change PCIe power control
echo -n on > /sys/bus/pci/devices/0000\:00\:01.0/power/control
sleep 1
# rescan for NVIDIA card (defaults to power/control = on)
echo -n 1 > /sys/bus/pci/rescan
# someone said that modprobe nvidia is needed also to load nvidia, to check
# modprobe nvidia
```
#### `disablegpu.sh`
```
#!/bin/sh

modprobe -r nvidia_drm
modprobe -r nvidia_uvm
modprobe -r nvidia_modeset
modprobe -r nvidia

# Change NVIDIA card power control
echo -n auto > /sys/bus/pci/devices/0000\:01\:00.0/power/control
sleep 1
# change PCIe power control
echo -n auto > /sys/bus/pci/devices/0000\:00\:01.0/power/control
sleep 1

# Lock system form loading nvidia module
mv /etc/modprobe.d/disable-nvidia.conf.disable /etc/modprobe.d/disable-nvidia.conf
```
### Create service locking NVIDIA GPU on shutdown

A service which locks GPU on shutdown / restart when it is not disabled by disablegpu.sh script is necessary. Otherwise, on next boot nvidia will be loaded (even if blacklisted with install command) and it won't be possible to unload them then.
```
sudo nano /etc/systemd/system/disable-nvidia-on-shutdown.service
```
```
[Unit]
Description=Disables Nvidia GPU on OS shutdown

[Service]
Type=oneshot
RemainAfterExit=true
ExecStart=/bin/true
ExecStop=/bin/bash -c "mv /etc/modprobe.d/disable-nvidia.conf.disable /etc/modprobe.d/disable-nvidia.conf || true"

[Install]
WantedBy=multi-user.target
```
Reload systemd daemons and enable the disable-nvidia-on-shutdown service:
```
 sudo systemctl daemon-reload
 sudo systemctl enable disable-nvidia-on-shutdown.service
```
Allow gpu to poweroff on boot
```
sudo nano /etc/tmpfiles.d/nvidia_pm.conf
```
```
w /sys/bus/pci/devices/0000:01:00.0/power/control - - - - auto
```
Finally check that everything is well configured:

Reboot and verify that nvidia is not loaded by running:
```
        lsmod | grep nvidia
```
Should not return anything

Disconnect / unplug charger and verify the power consumption with powertop is around 5W — FHD screen with no touchscreen — or 7.5W — 4K with touchscreen — on idle (`sudo powertop --auto-tune` previously launched).

Enable GPU by using the `enablegpu.sh` script

Check that the GPU is loaded by using:
```
        nvidia-smi
```
If good, launch unigine-valley with optirun:
```
        optirun unigine-valley
```
unigine-valley needs a GPU to work. Should activate the fans quickly.

Close all nvidia applications and disable gpu with the `disablegpu.sh` script
