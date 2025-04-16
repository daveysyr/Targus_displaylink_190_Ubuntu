# Targus_displaylink_190_Ubuntu
Installing a Targus Displylink 190 USB-C on Ubuntu


DisplayLink Installation on Ubuntu 24.10 (Manual Build)

This guide walks through building and installing DisplayLink driver v6.0.0 on Ubuntu 24.10 with kernel 6.x using the EVDI module from source.

Prerequisites

sudo apt update
sudo apt install dkms libdrm-dev libpciaccess-dev linux-headers-$(uname -r) build-essential

Step 1: Extract and Build

cd ~/Downloads
chmod +x displaylink-driver-6.0.0-24.run
./displaylink-driver-6.0.0-24.run --keep --noexec
cd DisplayLinkExtracted
tar -xzf evdi.tar.gz
cd evdi
make KVER=$(uname -r)

Step 2: Install Kernel Module and Library

sudo mkdir -p /lib/modules/$(uname -r)/kernel/drivers/gpu/drm/evdi
sudo cp module/evdi.ko /lib/modules/$(uname -r)/kernel/drivers/gpu/drm/evdi/
sudo depmod
sudo modprobe evdi initial_device_count=1

sudo cp library/libevdi.so.1.14.9 /usr/lib/libevdi.so.1
sudo ln -sf /usr/lib/libevdi.so.1 /usr/lib/libevdi.so

Step 3: Configure Module and Udev Rules

echo "options evdi initial_device_count=1" | sudo tee /etc/modprobe.d/evdi.conf

echo 'ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="17e9", ATTR{idProduct}=="6008", RUN+="/sbin/modprobe evdi"' \
  | sudo tee /etc/udev/rules.d/99-evdi.rules

Step 4: Preload evdi on Boot

sudo tee /etc/systemd/system/evdi-init.service > /dev/null << 'EOF'
[Unit]
Description=Load EVDI Kernel Module
After=systemd-modules-load.service

[Service]
Type=oneshot
ExecStart=/sbin/modprobe evdi

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable evdi-init.service

Step 5: Install DisplayLink Files

cd ~/Downloads/DisplayLinkExtracted
sudo mkdir -p /usr/lib/displaylink
sudo cp -r x64-ubuntu-1604/* /usr/lib/displaylink/
sudo cp displaylink-installer.sh /usr/lib/displaylink/
sudo cp service-installer.sh /usr/lib/displaylink/
sudo cp udev-installer.sh /usr/lib/displaylink/

Step 6: Create Systemd Service

sudo tee /usr/lib/systemd/system/displaylink-driver.service > /dev/null << 'EOF'
[Unit]
Description=DisplayLink Driver Service
After=graphical.target evdi-init.service

[Service]
ExecStart=/usr/lib/displaylink/DisplayLinkManager
Restart=always
StandardOutput=null

[Install]
WantedBy=graphical.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable displaylink-driver.service

Step 7: Finalise and Reboot

sudo update-initramfs -u
sudo reboot

Troubleshooting

If the second screen doesn't appear after reboot:

sudo systemctl restart displaylink-driver.service

Then check display status:

xrandr

