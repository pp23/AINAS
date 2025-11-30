# AI_NAS

This project builds a NAS with AI capabilities to classify automatically stored files and make them usable in a semantic search.

## Hardware

* [FriendlyElec CM3588 Plus](https://www.friendlyelec.com/index.php?route=product/product&path=60&product_id=299)
* Ordered via [Amazon seller WayPonDev](https://www.amazon.de/-/en/dp/B0DSGBPKC5?ref=ppx_yo2ov_dt_b_fed_asin_title&th=1) (32GB+64GB)
* Replaced the fan with a [Noctua NF-A4x10 5V](https://www.amazon.de/-/en/dp/B00NEMGCIA?ref=ppx_yo2ov_dt_b_fed_asin_title)
	* Cutting the edges 1mm in height made it possible to fit it in the metal case fixed with M3x10 screws
	* Noctua came with a 3pin->2pin adapter. The plug of the WayPonDev CPU fan has been used, soldered with the Noctua adapter.
* 1x [Samsung NVME 990 Pro 4TB](https://www.reichelt.de/de/de/shop/produkt/samsung_ssd_990_pro_4tb_m_2_nvme-384306) (3x remaining slots will get filled if the budget is available ;-) )
* [LAN cable is a CAT.6A](https://www.reichelt.de/de/de/shop/produkt/patchkabel_cat_6a_blau_0_25_m-198830) with up to 10GBps in theory

### Updating Samsung 990 Pro Firmware (not working yet, but should with VFIO Kernel module)

* Login into your NAS
* Download the latest firmware update iso from [Samsung Storage Firmware](https://semiconductor.samsung.com/consumer-storage/support/tools/)
  ```
  wget https://download.semiconductor.samsung.com/resources/software-resources/Samsung_SSD_990_PRO_7B2QJXD7.iso
  ```
* Mount the iso-image:
  ```
  sudo mkdir -p /mnt/iso
  sudo mount -o loop ./Samsung_SSD_990_PRO_7B2QJXD7.iso /mnt/iso
  ```
* Run the update image in a x86 virtual machine:
  ```
  qemu-system-x86_64 -kernel /mnt/iso/bzImage \
                     -initrd /mnt/iso/initrd \
                     -drive file=disk.img,format=raw,if=virtio \
                     -m 2G \
                     -nographic \
                     -append "root=/dev/vda rw console=ttyS0,115200 earlyprintk=serial earlycon"
  ```
  * this should show the kernel boot logs and ends in Samsungs fumagician cli nvme update tool
  * because no NVME is available in the VM, the tool does not find any NVME
* Make the NVME available in the VM (in theory, not tested yet due to missing VFIO kernel module)
  ```
  # unbind the nvme:
  lspci -nn | grep -i nvme
  # outputs: 0003:31:00.0
  # use the pci address in the unbind command:
  echo 0003:31:00.0 | sudo tee /sys/bus/pci/devices/0003\:31\:00.0/driver/unbind
  ```
* Bind it to the VFIO driver (works only, if IOMMU is supported!)
  ```
  sudo modprobe vfio
  sudo modprobe vfio-pci
  # bind the device:
  echo 144d a808 | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id
  # Use your device’s vendor & device ID (from lspci -nn).
  # Check binding:
  lspci -k -s 0003:31:00.0
  # should show: Kernel driver in use: vfio-pci
  ```
* Add the NVMe to QEMU:
  ```
  qemu-system-x86_64
  [...]
   -device vfio-pci,host=0000:03:00.0,multifunction=on
  ```

## Software

* [OpenMediaVault with Linux version 6.1.99 from FriendlyElec](https://download.friendlyelec.com/CM3588)
* zfs-2.3.1-1~bpo12+1 from Debian backports sources based on the [OpenZFS installation guide](https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/index.html#installation)
	* Required [compiliation of Linux headers for DKMS](https://github.com/DukeChocula/CM3588?tab=readme-ov-file#compiling-linux-headers-for-dkms-dynamic-kernel-module-support)

### ZFS

* Created an encrypted zfs-pool "naspool" with lz4 compression according to [Alpine Wiki](https://wiki.alpinelinux.org/wiki/ZFS)
	* encryption key is stored in /etc/naspool.key in the root-fs of the NAS (a [backup](./naspool.key) can be found on WOPR)

```bash
# create the encryption key
sudo dd if=/dev/random of=/etc/naspool.key bs=32 count=1
sudo chmod 0600 /etc/naspool.key

# create the zfs pool
sudo zpool create -o ashift=12 -O acltype=posixacl -O compression=lz4 -O dnodesize=auto -O normalization=formD -O relatime=on -O xattr=sa -O encryption=aes-256-gcm -O keylocation=file:///etc/naspool.key -O keyformat=raw -O mountpoint=none naspool nvme3n1
```
