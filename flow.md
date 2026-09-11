# How Yocto WIC Creates a Disk Image

I wanted to understand the WIC flow properly instead of just using the `.wks` file without knowing what happens behind it.

So I started tracing the source from BitBake until the final `.wic` image is created.

---

## 1. `image_types_wic.bbclass`

I started from:

```text
openembedded-core/meta/classes-recipe/image_types_wic.bbclass
```

This is the class responsible for creating the WIC image.

First, it determines which WKS file should be used.

Then it searches for the file in the WKS search paths and gets its full path.

For the BeagleV-Ahead, the WKS file is:

```text
meta-riscv/files/wic/beaglev-ahead.wks
```

After the `.bbclass` finds the WKS file, it starts WIC using a command similar to:

```bash
wic create <wks-file> ...
```

It also passes the paths and files WIC needs, such as the rootfs, kernel files, work directory, output directory, and native sysroot.

So at this point the flow is:

```text
BitBake  >  image_types_wic.bbclass   >  find the WKS file  >   wic create
---

## 2. What the WKS File Actually Does

The WKS file itself does not create the disk image.

It only describes the layout we want.

For example:

```wks
bootloader --ptable gpt

part --label empty --part-name empty --align 4096 --size=2M

part /boot     --source bootimg-partition     --fstype=ext4     --label boot     --part-name boot     --align 4096     --size=100M

part /     --source rootfs     --fstype=ext4     --label root     --part-name root     --align 4096     --size=1G
```

So the WKS is mainly describing things like:

- partition table type,
- partition size,
- alignment,
- filesystem type,
- source,
- labels,
- partition names.

I think of it as:

```text
WKS = description of the disk layout
```

It is not the code that physically creates the disk.

---

## 3. The `wic` Executable

After that, I checked the `wic` executable itself.

I found that it is mainly a launcher.

It calls:

```python
from wic.cli import main
main()
```

So the next file is:

```text
wic/cli.py
```
---

## 4. `cli.py`

Inside `cli.py`, WIC reads the arguments that were passed to it.

It gets information such as:

- WKS file path,
- work directory,
- output directory,
- image name,
- kernel directory,
- rootfs directory,
- native sysroot,
- imager type.

After that it calls:

```python
engine.wic_create(...)
```

So now the flow becomes:

```text
wic  >  cli.py  >  engine.wic_create()
```

---

## 5. `engine.py`

At first I was a little confused by `engine.py`.

What I understood is that `engine.py` does not build the disk image itself.

Its job is to decide which imager plugin will build the image.

The default imager is:

```text
direct
```

So `engine.py` looks for the imager plugin called `direct`.

It finds:

```text
DirectPlugin
```

inside:

```text
wic/plugins/imager/direct.py
```

Then it creates the plugin and calls:

```python
plugin.do_create()
```

The easiest way I understand this part is:

```text
cli.py
= reads the request

engine.py
= decides who will handle the request

DirectPlugin
= starts creating the actual disk image
```
---

## 6. `DirectPlugin`

Inside `direct.py`, one of the important lines is:

```python
self.ks = KickStart(wks_file)
```

This is where the WKS file starts being read and parsed.

`KickStart()` takes us to:

```text
wic/ksparser.py
```

---

## 7. `ksparser.py`

`ksparser.py` opens the WKS file and reads it line by line.

Its job is to understand the WKS instructions and convert them into Python information.

For example:

```wks
bootloader --ptable gpt
```

becomes something equivalent to:

```text
ptable = gpt
```

And every line starting with:

```wks
part
```

is converted into a `Partition` object.

For example:

```wks
part /boot --source bootimg-partition --fstype=ext4 --align 4096 ...
```

becomes information like:

```text
mountpoint = /boot
source     = bootimg-partition
fstype     = ext4
align      = 4096
size       = ...
```

So this part is important:

```text
ksparser.py
= reads and understands the WKS

it does not create the disk yet
```

---

## 8. `partition.py`

After the WKS is parsed, every `part` line becomes a `Partition` object.

The partition object stores information such as:

```text
size
align
offset
source
fstype
label
part-name
no-table
```

It also helps prepare the content needed for each partition.

For example:

```text
--source rootfs
```

uses the rootfs source plugin.

And:

```text
--source bootimg-partition
```

uses the boot image partition source plugin.

So `partition.py` prepares and stores the partition information, but it still does not decide the final physical position on the disk.

---

## 9. Back to `direct.py`: `PartitionedImage`

After the WKS is parsed and the partition information is ready, the flow goes back to `direct.py`.

Here there is an important class called:

```text
PartitionedImage
```

This is where the actual disk layout starts being calculated.

The main flow inside it is:

```text
prepare()  >  layout_partitions()  >  create()  >  assemble()
---

## 10. `prepare()`

`prepare()` prepares the content of every partition.

For example:

- root filesystem,
- boot filesystem,
- other partition data.

It also calculates the final size of the partition and converts it into sectors.

So before WIC decides where a partition goes, it first needs to know how big it is.

---

## 11. `layout_partitions()`

This is the part that calculates where every partition will be placed on the disk.

It handles things like:

- start sector,
- partition size,
- alignment,
- offset,
- total image size.

One important thing I found here is:

```python
GPT_OVERHEAD = 34
```

When the partition table is GPT, WIC reserves the first 34 sectors for the normal GPT structure.

With 512-byte sectors, the normal GPT layout is:

```text
LBA 0       Protective MBR
LBA 1       Primary GPT Header
LBA 2-33    Primary GPT Entry Array
LBA 34      First usable sector
```

So WIC starts its normal GPT layout calculation after sector 34.

---

## 12. Why `--align 4096` Gives LBA 8192

The BeagleV-Ahead WKS uses:

```wks
--align 4096
```

This means:

```text
4096 KiB = 4 MiB
```

With 512-byte sectors:

```text
4 MiB / 512 = 8192 sectors
```

WIC starts from sector 34 and then moves to the next valid 8192-sector alignment boundary.

So the first partition starts at:

```text
LBA 8192
```

This matches the generated WIC image I inspected.

---

## 13. `create()`

After the layout is calculated, `create()` creates the actual disk image.

WIC first creates the image file.

Then it uses GNU `parted` to create the partition table.

For GPT it runs something equivalent to:

```bash
parted -s IMAGE mklabel gpt
```

Then it creates the partitions using commands like:

```bash
parted -s IMAGE unit s mkpart ...
```

It also uses `sfdisk` for things such as:

- partition labels,
- partition UUIDs,
- disk GUID,
- partition types.

So WIC is not manually writing the whole GPT structure byte by byte.

It uses tools such as:

```text
parted
sfdisk
```

to create and configure the real partition table.

---

## 14. `assemble()`

After the disk and partitions are created, `assemble()` copies the prepared partition contents into the correct places inside the image.

The destination is based on:

```text
partition start sector × sector size
```

For example, if a partition starts at:

```text
LBA 16384
```

and the sector size is:

```text
512 bytes
```

then:

```text
16384 × 512 = 8 MiB
```

So the partition content is written starting at 8 MiB inside the final disk image.

---

# What I Understood From This

The main thing I learned is that the WKS file itself does not create the image.

Each part has a different job:

```text
image_types_wic.bbclass
    finds the WKS file and starts WIC

WKS
    describes the disk layout

cli.py
    reads the WIC command and arguments

engine.py
    selects the imager plugin

DirectPlugin
    manages the disk-image creation flow

ksparser.py
    reads and parses the WKS

Partition
    stores and prepares partition information

PartitionedImage
    calculates the physical disk layout

parted / sfdisk
    create and configure the actual partition table

assemble()
    copies the prepared partition contents into the final image
```

One important discovery for the BeagleV-Ahead work was:

```python
GPT_OVERHEAD = 34
```

This explains why normal WIC expects the standard GPT structure at the beginning of the disk:

```text
LBA 0       Protective MBR
LBA 1       GPT Header
LBA 2-33    GPT Entry Array
```

This became important later because the BeagleV-Ahead boot layout I was testing needed a different placement for some of the bootloader and GPT data.
