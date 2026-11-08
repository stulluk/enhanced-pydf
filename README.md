# enhanced-pydf

A small extension of **[pydf](https://github.com/k4rtik/pydf)** — the colourised `df(1)`-style disk usage viewer — for **Linux** systems. The upstream project shows filesystem, sizes, and mount points; this fork adds **physical disk identity**, **bus / transport type**, and **nominal link speed** so you can see *what* is behind each mount at a glance.

## Original goal

I have used the classic `pydf` package (also on [PyPI](https://pypi.org/project/pydf/)) for years on Ubuntu and Debian. It is pleasant and readable, but when many disks and partitions are mounted I still had to cross-check **`lsblk`**, **`/sys`**, or labels to remember **which physical drive** each line referred to, and **how it was connected** (NVMe vs SATA vs USB, and roughly how fast the link is). This script keeps the same spirit as pydf while folding that context into the table.

## What is new compared to upstream pydf

| Feature | Upstream pydf | enhanced-pydf |
|--------|----------------|---------------|
| Disk vendor / model | No | **Yes** — `Disk` column (from `lsblk` + sysfs, with udev `\xNN` decoding) |
| Bus / interface | No | **Yes** — `Bus` column (`lsblk` **TRAN**: NVMe, SATA, USB, …) |
| Link speed / generation | No | **Yes** — `Link max` column |
| NVMe PCIe info | — | **Gen × lanes** from sysfs (`max_link_speed` / `max_link_width`) plus a **static ~GB/s hint** (approximate one-way theoretical data rate) |
| SATA speed | — | **From sysfs** (`sata_spd` per ATA link), not a blind “6 Gb/s” guess — e.g. SATA II shows **3.0 Gbps** when the kernel reports it |
| USB | — | Rough class speed from sysfs `speed` (Mbps) where available |
| PCI BDF | — | **Optional** — **`--pci-bdf`** (or `show_pci_bdf = True` in `pydfrc`) adds **`PCI BDF`** after **Disk** to tell identical drives apart (NVMe `address` or last BDF in the block device path) |
| Sort order | Mount path only | **`--sort-by-filesystem`** sorts by block device name, then mount point |
| Usage bar | Same width on every row | **Length proportional to capacity** — the largest filesystem gets the full bar, smaller ones are drawn shorter |
| Column layout | Default | **`Disk`**, **`Bus`**, **`Link max`** after **Mounted on**; **`PCI BDF`** only when enabled (configurable via `~/.pydfrc` like upstream) |
| Opt out | — | **`--no-disk`** removes the extra disk-related columns (classic layout) |
| Version string | `12` | `12+disk+bus` |

**Performance:** One `lsblk -P` call per backing block device (cached per device), plus small sysfs reads. SATA link resolution uses `scsi_host` **`port_no`** or **`unique_id`** to map to the correct `linkN`; if that fails and only one ATA link exists on the controller, that link’s speed is used as a fallback.

**Caveats:** Remote filesystems (NFS, CIFS, …) and non-block-backed entries show `-`. Stacked devices (LVM, dm-crypt, …) may show incomplete bus/speed information. The **~GB/s** values for PCIe are **rounded static estimates**, not measured throughput.

## Requirements

- Linux (paths and tools assume a typical util-linux + sysfs layout).
- Python 3.x. The script starts with `#!/usr/bin/python3`, because current distributions no longer ship `/usr/bin/python`. Python 2.7 remains the upstream baseline, but it must be started explicitly: `python2 pydf`.
- `lsblk` from **util-linux** (for `VENDOR`, `MODEL`, `TRAN`, `PKNAME`).

## Installation

```bash
# Example: install into your own PATH
cp pydf ~/bin/pydf
chmod +x ~/bin/pydf
```

You can also run it from a clone: `./pydf`.

## Usage

```bash
pydf                      # default: Disk, Bus, Link max (unless disabled in pydfrc)
pydf --pci-bdf            # also show PCI BDF after Disk
pydf --sort-by-filesystem # sort by block device, then mount point
pydf --no-disk            # same column set as classic pydf (no extra columns)
pydf --bw                 # no colours (good for logs or README captures)
pydf -v                   # version: 12+disk+bus
```

Configuration via **`/etc/pydfrc`** or **`~/.pydfrc`** works like upstream (`format`, colours, etc.). If your `format` omits the new columns, **`disk`**, **`iface`**, and **`speed`** are **re-injected** after **`on`** / **`disk`** unless you pass **`--no-disk`**. **`PCI BDF`** is only injected when **`--pci-bdf`** is set or **`show_pci_bdf = True`** in `pydfrc`; without that, any **`pci`** column in `format` is stripped.

## Capacity-scaled usage bars

The bar length follows the filesystem size: the largest listed filesystem takes the full bar width, and every other bar is shortened by the same ratio, so one `#` stands for the same amount of space on every row. The reference is the largest **listed filesystem** (partition), not the physical disk behind it. With `-i`, inode counts are compared instead of bytes. A very small filesystem still gets one cell (`[.]`), and the bar resolution depends on the terminal width.

```text
Filesystem      Size  Used Avail Use%                                      Mounted on
/dev/nvme2n1p6  257G  166G   79G 64.4 [###..]                              /
/dev/nvme1n1p2 1073M   11M 1062M  1.0 [.]                                  /boot/efi
/dev/nvme2n1p5  657G  157G  498G 24.0 [###.........]                       /media/DATA
/dev/sda1       504G  172G  332G 34.0 [###......]                          /media/G550
/dev/sda2       412G  349G   62G 84.7 [#######.]                           /media/H450
/dev/nvme1n1p3  680G  564G  115G 82.9 [###########..]                      /media/KINGDATA
/dev/nvme0n1    916G  710G  204G 77.5 [#############....]                  /media/NEW1TB
/dev/nvme1n1p1  234G  114G  117G 48.7 [##..]                               /media/UBUNTU24
/dev/nvme3n1   1833G 1545G  284G 84.3 [#############################.....] /media/WORK
efivarfs        256k   71k  180k 27.7 [.]                                  /sys/firmware/efi/efivars
```

## Output comparison (same machine, clean `HOME`, `--bw`)

These samples compare the extra columns and were captured before capacity-scaled bars were added.

The following samples were produced on one developer machine with an empty `HOME` so no personal `pydfrc` overrides the default column layout.

### Upstream pydf (k4rtik/pydf, unmodified)

```text
Filesystem     Type      Size  Used Avail Use%         Mounted on              
/dev/nvme2n1p6 ext4      257G   12G  233G  4.5 [.....] /                       
/dev/nvme1n1p2 vfat     1073M   11M 1062M  1.0 [.....] /boot/efi               
/dev/nvme2n1p5 ext4      657G  363G  292G 55.3 [###..] /media/DATA             
/dev/sda1      ext4      504G  182G  322G 36.0 [##...] /media/G550             
/dev/sda2      ext4      412G  308G  103G 74.8 [####.] /media/H450             
/dev/nvme1n1p3 ext4      680G  447G  232G 65.7 [###..] /media/KINGDATA         
/dev/nvme0n1   ext4      916G  636G  278G 69.4 [###..] /media/NEW1TB           
/dev/nvme3n1   ext4     1833G 1693G  136G 92.4 [#####] /media/WORK             
efivarfs       efivarfs  128k   67k   56k 52.5 [###..] /sys/firmwar~efi/efivars
```

### enhanced-pydf (this repository)

```text
Filesystem     Type      Size  Used Avail Use%         Mounted on                Disk                         Bus  Link max                
/dev/nvme2n1p6 ext4      257G   12G  233G  4.5 [.....] /                         Samsung SSD 980 PRO 1TB      NVMe PCIe Gen4 x4 (~7.9 GB/s)
/dev/nvme1n1p2 vfat     1073M   11M 1062M  1.0 [.....] /boot/efi                 KINGSTON SNV3S1000G          NVMe PCIe Gen4 x4 (~7.9 GB/s)
/dev/nvme2n1p5 ext4      657G  363G  292G 55.3 [###..] /media/DATA               Samsung SSD 980 PRO 1TB      NVMe PCIe Gen4 x4 (~7.9 GB/s)
/dev/sda1      ext4      504G  182G  322G 36.0 [##...] /media/G550               ATA WDC WD10JPVT-75A1YT0     SATA 3.0 Gbps (SATA II)      
/dev/sda2      ext4      412G  308G  103G 74.8 [####.] /media/H450               ATA WDC WD10JPVT-75A1YT0     SATA 3.0 Gbps (SATA II)      
/dev/nvme1n1p3 ext4      680G  447G  232G 65.7 [###..] /media/KINGDATA           KINGSTON SNV3S1000G          NVMe PCIe Gen4 x4 (~7.9 GB/s)
/dev/nvme0n1   ext4      916G  636G  278G 69.4 [###..] /media/NEW1TB             KINGSTON SNV3S1000G          NVMe PCIe Gen4 x4 (~7.9 GB/s)
/dev/nvme3n1   ext4     1833G 1693G  136G 92.4 [#####] /media/WORK               Samsung SSD 970 EVO Plus 2TB NVMe PCIe Gen3 x4 (~3.9 GB/s)
efivarfs       efivarfs  128k   67k   56k 52.5 [###..] /sys/firmware/efi/efivars -                            -    -                       
```

*(Your paths, disk models, and link lines will differ; the extra columns follow the same rules everywhere.)*

## Upstream / credits

- Original **pydf** is by **Radovan Garabík**; packaging and Git history for the classic project: [k4rtik/pydf](https://github.com/k4rtik/pydf), [PyPI pydf](https://pypi.org/project/pydf/).
- This repository contains only the **enhanced single-file script** and documentation; it is not affiliated with the upstream maintainers.

## License

The original program is in the **public domain** (see upstream `COPYING`). Enhancements in this repository are also placed in the **public domain** to match.
