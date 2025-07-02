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
