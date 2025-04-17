

# DisplayLink + EVDI + DKMS + Secure Boot Setup (Ubuntu)

This guide documents how to install the DisplayLink driver with EVDI support on Ubuntu with Secure Boot enabled and persistent support across kernel updates via DKMS.

Tested on:  
- **Ubuntu 24.04+**
- **Kernel:** `6.11.0-24-generic`
- **DisplayLink Driver:** 6.0.0-24  
- **EVDI version:** 1.14.1

---

## ⚙️ Prerequisites

```bash
sudo apt update
sudo apt install dkms make gcc linux-headers-$(uname -r) openssl mokutil

📦 Extract DisplayLink Package

cd ~/Downloads
chmod +x displaylink-driver-6.0.0-24.run
./displaylink-driver-6.0.0-24.run --noexec --target DisplayLinkExtracted

📁 Build and Install EVDI

cd DisplayLinkExtracted
tar -xf evdi.tar.gz
sudo cp -r evdi /usr/src/evdi-1.14.1

📝 Create /usr/src/evdi-1.14.1/dkms.conf

PACKAGE_NAME="evdi"
PACKAGE_VERSION="1.14.1"
BUILT_MODULE_NAME[0]="evdi"
BUILT_MODULE_LOCATION[0]="module"
DEST_MODULE_LOCATION[0]="/kernel/drivers/gpu/drm/evdi"
MAKE[0]="make -C module KVER=${kernelver} DKMS_BUILD=1"
CLEAN="make -C module clean"
AUTOINSTALL="yes"

🛠 Modify Makefile at /usr/src/evdi-1.14.1/module/

Ensure this block exists (or override the whole Makefile):

obj-m := evdi.o
evdi-objs := evdi_platform_drv.o evdi_platform_dev.o evdi_sysfs.o evdi_modeset.o \
             evdi_connector.o evdi_encoder.o evdi_drm_drv.o evdi_fb.o evdi_gem.o \
             evdi_painter.o evdi_params.o evdi_cursor.o evdi_debug.o evdi_i2c.o \
             evdi_ioc32.o

all:
	$(MAKE) -C /lib/modules/$(KERNELRELEASE)/build M=$(PWD) modules

clean:
	$(MAKE) -C /lib/modules/$(KERNELRELEASE)/build M=$(PWD) clean

🔐 Sign Kernel Module for Secure Boot

mkdir -p ~/kernel-signing && cd ~/kernel-signing
openssl req -new -x509 -newkey rsa:2048 -keyout MOK.key -out MOK.crt -nodes -days 36500 -subj "/CN=Module Signer/"
openssl x509 -outform DER -in MOK.crt -out MOK.der
sudo mokutil --import MOK.der

📌 Reboot and enrol the key via the blue MOK manager screen.
🧩 Build + Install with DKMS

sudo dkms add -m evdi -v 1.14.1
sudo dkms build -m evdi -v 1.14.1
sudo dkms install -m evdi -v 1.14.1

You should see:

evdi.ko:
  Running module version sanity check.
  Original module
  No changes.
  Signing module /lib/modules/.../evdi.ko

📡 Enable DisplayLink Driver

sudo systemctl enable displaylink-driver.service
sudo systemctl start displaylink-driver.service

✅ Verify

lsmod | grep evdi
xrandr

You should see evdi loaded and your DisplayLink screen connected (DVI-I-1 or similar).
🔁 Persistence Across Kernel Updates

Because of DKMS and the signing setup, the EVDI module will now:

    Automatically rebuild for new kernels

    Be signed with your enrolled MOK key

    Load at boot with Secure Boot enabled

🔄 Optional: Reset DKMS

sudo dkms remove -m evdi -v 1.14.1 --all
