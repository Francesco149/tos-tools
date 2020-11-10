# NOTE
most of these tools are incomplete, undocumented, old, unmaintained.
the only two scripts I have been using and updating are tos-mirror and tos-commit

# intro
various tools I glued together for my TempleOS workflow

these are meant for linux but should run fine on most unixes

# credits
* fuse red sea implementation by: https://github.com/obecebo/redseafs
* TOSZ written by Terry A. Davis himself

# requirements
* rsync (for tos-mirror and tos-export)
* fuse (for isoc-mount)
* python2.7 (for isoc-mount)
* fusepy for python2.7 (for isoc-mount)
* g++ (for TOSZ)
* ffmpeg (to convert screencasts to mp4)
* qemu with kvm support (for the qemu scripts)

on arch linux, you would install these like so:

```sh
sudo pacman -S fuse python2 python2-pip rsync g++ ffmpeg
pip2.7 install --user fusepy
sudo modprobe fuse
```

my scripts assume that the FAT TempleOS partition is mounted at /mnt/tos,
so make sure to have an /etc/fstab entry like this:

```fstab
/dev/sda2 /mnt/tos vfat user,owner,rw,umask=000   0 0
```

and make sure /mnt/tos exists

```sh
mkdir -p /mnt/tos
```

# install
```sh
git clone https://github.com/Francesco149/tos-tools
cd tos-tools
g++ TOSZ.CPP -o TOSZ
echo "export PATH=\"\$PATH:$(pwd)\"" >> ~/.bashrc
```

if you don't use bash, just add the path to your particular shell's profile
instead

# usage
* ![tos-mirror: copy files from red sea iso's to your TempleOS partition](#tos-mirror)
* ![isoc-mount: mount red sea iso's](#isoc-mount)
* ![TOSZ: terry's utility to uncompress and convert TempleOS files](#TOSZ)
* ![tos-export: exports your install for git](#tos-mirror)
* ![iso-mount: mounts a non-redsea iso with case-sensitive filenames](#iso-mount)
* ![tos-qemu: creates and runs a qemu virtual machine with a TempleOS iso](#tos-qemu)

# tos-mirror
this script mounts redsea iso's (often named .ISO.C but sometimes .ISO)
which are all the most recent TempleOS ISO's and copies the contents
to your TempleOS FAT32 partition.

it automatically excludes .BIN.C files so it doesn't overwrite your
bootloader or kernel which would break your install.

this overwrites without prompting, so if you're not sure, run it with
```--dry-run``` first to see what it would copy

I use this to install supplemental ISO's and stuff like that without having
to burn them

note that this assumes that your TempleOS fat partition is mounted at
```/mnt/tos``` . if it isn't, export TOS_MOUNTPOINT in your bashrc to
change it

merging the contents of the iso with your root TempleOS directory

```
tos-mirror TempleOSCDRS.ISO
```

copying to a specific directory (creates it if it doesn't exist)

```
tos-mirror TOS_Supplemental1.ISO.C Home/Sup1
```

checking what files would be copied without actually copying them

```
tos-mirror TOS_Web.ISO.C --dry-run
```

after you're done copying everything you can boot into TempleOS and do a

```
CopyTree("D:/","C:/");
```

then recompile the kernel and reinstall the bootloader (if you copied any
kernel or compiler changes)

```
BootHDIns('C');
```

make sure to fill in the correct I/O ports or you will brick your install.
I'll eventually write HolyC program to programmatically run BootHDIns and
enter pre-configured I/O ports

as a safety measure, you could instead boot into D (the FAT partition),
recompile the kernel from there, reboot, see if it still boots and then
copy it over to redsea. big kernel changes can cause compile errors due
to having to bootstrap new symbols, so be careful when copying over an
entirely different version of the OS

this script relies on not having any other RedSea mounts active. if it
finds RedSea mounts in /etc/mtab it will try to umount ~/iso and wait
until it disappears from mtab.

it also cleans up after itself, so you don't have to umount the iso after
it's done

# isoc-mount
this is a fuse implementation of the red sea filesystem made by @obecebo .
it's used by tos-mirror to mount the iso's


```
isoc-mount TOS_Supplementals1.ISO.C ~/iso
cd ~/iso
ls
```

# TOSZ
terry's utility to uncompress and convert TempleOS files so they're usable
from other OSes

uncompress SomeFile.HC.Z in place, convert non-standard ascii and
rename to SomeFile.HC

```
TOSZ -ascii SomeFile.HC.Z
```

call ffmpeg to convert MV files and an AU file to a combined MP4 file

```
TOSZ -mp4 VID%03d.MV AUDIO.AU MOVIE.MP4
```

# tos-export
this assumes you have my TOSToGit program that exports your entire install
for consumption on other OSes

this script simply syncs the Wb and Git directories in your TempleOS
partition to ~/TempleOSWb and ~/TempleOSGit, taking care to delete any
files that have been deleted from the source

this is supposed to be used in conjuction with git to keep track of changes
you make

when you're done playing with TempleOS, before booting back into linux
you run

```cpp
CopyTree("C:/","D:/");
FreshenSite;
FreshenBlog;
FreshenGit;
```

CopyTree is actually just for syncing the two partitions, it's not
necessary to run it for the exporting

you can put these commands in a function exported in your home or in a
.HC.Z file and #include it to save typing

then once you boot back into linux you would run

```
tos-export
cd ~/TempleOSGit
git add -u
git commit -m "did some stuff"
git push
```

or if you need to split changes into smaller commits, just use
```git add -p```

at the moment I don't keep the site on git but you can do the same with it

# iso-mount
the non-redsea iso's require some weird mount options to get case sensitive
filenames. this is a simple wrapper for mount which passes those params
and mounts to /mnt/iso

ignore warning messages about /mnt/iso not being mounted

```
iso-mount file.ISO
```

# tos-qemu
creates and runs a qemu virtual machine for a TempleOS iso. works all the
way back to LoseThos 7.06 (LTCD_2011-09-24T18:53:22.ISO 3f03b65275d867617fa1ef36535e6399adc2a2bd)

I use this to install and archive historical versions of TempleOS

```
tos-qemu TOS_Distro.ISO
```

this does a few things:
* renames the .iso by removing characters unsupported by qemu like `:`
* creates a qcow2 virtual disk at <path to iso>/<iso name>.iso.qcow2 if
  it doesn't exist
* if it's the first boot, it mounts the iso, otherwise it boots from disk
* if it's NOT the first boot, it first mounts the vm's image to ~/vmtos,
  copies all files in ~/.tosfiles to the second partition  and umounts it.
  then it finally boots. note that this assumes that your 2nd partition
  in the vm is FAT32. if ~/.tosfiles doesn't exist this step is skipped

the script is configured to give 2 gigs of ram and 4 cores to the vm's, as
well as 500MB of disk space. edit the script if you need to change these

for versions of TempleOS/LoseThos that didn't officially support qemu,
you can just use the I/O ports for the automatically probed ata and it will
work

there's also a mode where it only mounts or unmounts the virtual drive:

```
tos-qemu TOS_Distro.ISO mount
tos-qemu TOS_Distro.ISO umount
```

only works if the virtual machine has already beenc created
