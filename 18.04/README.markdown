# Kickstart configs for Ubuntu 18.04

- **Status**: beta (expect to be production a little after 18.04.1 comes out)

This folder contains kickstart configs designed for Ubuntu 18.04

## Notable changes since 16.04

See [release notes](https://wiki.ubuntu.com/BionicBeaver/ReleaseNotes) for a complete list but some things worth noting.

- Once again python 2.7 is not installed by default but this is also the last LTS release that will include python 2.7 in main. Also noticed no symlink is created for `python`. That means you must call python using `python3`.
- Networking is now using [netplan.io](https://netplan.io/) for network interfaces and `systemd-resolved` for dns.
- While these kickstarts don't install it by default `chrony` is the new recommended ntp server.  When no server is installed the system will make use of the built-in `systemd-timesyncd.service` which should keep it pretty close to accurate.

### Kickstart changes

Changes made to kickstart file compared to last release

- mount `/dev/shm` with `rw,noexec,nodev,nosuid` to help prevent malware from getting installed.  Note that Ubuntu suggests using `/run/shm` but by default `/run/shm` symlinks to `/dev/shm`.
- Change the console screen size to `1024x768x24`.  If this causes issues comment out the `add-kernal-opts` line towards top of kickstart.

## Getting started

- With 18.04 the default server iso is a live cd using Ubuntu's new install gui. Several features like lvm are not available with that image and so no testing has been performed.  Testing has been done with the [alternate install image](http://cdimage.ubuntu.com/releases/18.04/release/).  The netboot image should also work.
- You will need a method of hosting the file.  Most common options are via a web server or nfs server.  This is beyond the scope of this project but there's no special configuration needed.  Just place the kickstart file you want in the nfs or web server and enter that address at boot.
- The way kickstart is loaded is it first obtains an IP over dhcp.  At that point it connected to remote server specified with `ks=` option.  Resets networking and goes by what it says in kickstart file.  This means for things to Just Work out of the box your local network must have dhcp even if it's only a few ips with really short expire times.
- These instructions and kickstart are designed to install a virtual machine.  Physical servers can be more tricky to determine the names of partitions and in general varies too much to create a standard kickstart file.

## Usage

- Make any modifications you wish to the config.
    - the initial username and password
    - if you setup an `apt-cacher-ng` server, uncomment those lines and update the server address.  Remember it's specified in two places.  Once towards the top which is used by installer and one in `%post` section which sets it up to be used in final system
    - alter the partition layout if you wish
    - add any more packages you wish to install
- Create the VM just make sure the drive is at least 10GB
    - for vmware users, you can create the network interface using `vmxnet3` as that is supported in the kernel
- Attach the OS ISO to cdrom and make sure to activate it on boot
- Startup VM
- You'll first be prompted to choose language, go ahead and choose something and hit enter (note that the kickstart file sets language of system)
- Press `F6` and then hit `ESC` and this will bring you to boot line.
- At the end of the line add `ks=http://your-server.example.com/ks-1804minimalvm.cfg` or whatever you named file.
- Press `ENTER` to start the installation

## Configuration

Here's what gets setup based on the distro and specific kickstart file you use.

### ks-1804minimalvm.cfg

This is the classic kickstart file that I've created with every LTS version.

- Requires minimum 10GB disk
- Initial user is `ubuntu` with password `ChangeMe`
- Installs the following
    - bash-completion
    - curl
    - latest version of git using [Ubuntu Git Maintainer's PPA](https://launchpad.net/~git-core/+archive/ubuntu/ppa)
    - man
    - net-tools (includes commands like `netstat` and `ifconfig`)
    - openssh-server
    - vim
    - wget
- Packages that are listed but commented out
    - chrony - will give more accurate time than timesyncd but not needed
    - haveged - seeds `/dev/random` with more entropy sources. This is useful with VMs where it's hard to get a lot of entropy.  Most of the use cases can get by without this but if you are running a service that uses a lot of encryption consider adding this
    - open-vm-tools - only useful if vm is running on VMWare and you don't plan to install VMWare's tools.
- I install git from the PPA since I generally want the latest version of git and that PPA is managed by the same people that manage the git package in Ubuntu so I trust them.
- Set [XDG Base Directories](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html).  While it shouldn't be needed unless different from default I've had programs not write to the proper directory unless these variables exist.
- Set vim background to dark.
- Change default umask from `022` to `027`, meaning files created are not world readable.
- Turns off installation of recommended packages.
- Enables automatic security updates.
- All partitions are `EXT4` as this is still the default filesystem.
- All partitions except `/boot` are in a LVM.
- After install, with 10GB disk, leaves about 2GB left to allocate to whatever partition you want using `lvextend`
- Creates several partitions.  Reason I made so many is to hopefully be left with as little stuff in `/` partition as possible.  Notably I don't create a `/tmp` partition although this is common.  If you are removing partitions I highly recommend keeping `/home` and `/var/log` since a lot of times those can fill up fast.  Depending on what you are using it for and where it gets installed you may want to create a `/opt` or `/srv`.  Although I would recommend creating a second drive for those.  Then if something is messed up you can remount that drive somewhere else and get the data.  I also add various options to mounts.  I add `noatime` to all so it doesn't write access times to files (note that `defaults` will add `relatime` if you don't specify `noatime`).  Then add `nodev` and `noexec` where I can for security.  See below for partition layout.  I then mount `/dev/shm` with `nosuid,nodev,noexec` since it's a common place exploits are placed.

| mount    | size  | options                |
| -------- | ----- | ---------------------- |
| /boot    | 512MB | `noatime,nodev`        |
| /        | 1GB   | `noatime`              |
| /usr     | 2GB   | `noatime,nodev`        |
| /var     | 1.5GB | `noatime,nodev`        |
| /var/log | 512MB | `noatime,nodev,noexec` |
| /home    | 512MB | `noatime,nodev`        |
| swap     | 2GB   |                        |
| /dev/shm | tmpfs | `nosuid,nodev,noexec` |

- One note on partitions. Starting in version 17.04 it was [announced](http://blog.surgut.co.uk/2016/12/swapfiles-by-default-in-ubuntu.html) that by default on non-lvm volumes a swapfile will be created instead of a swap partition.  Reasoning is it's easier to adjust the size of a swapfile later vs repartitioning disk.  For LVM it creates a swap logical volume inside the volume group.  This is because when using snapshots a changing swapfile will increase size of snapshot.  This is why it doesn't use a swapfile.
- Output from `df -h` and `vgs`

```
Filesystem               Size  Used Avail Use% Mounted on
udev                     482M     0  482M   0% /dev
tmpfs                     99M  624K   98M   1% /run
/dev/mapper/vg0-root     945M  171M  710M  20% /
/dev/mapper/vg0-usr      1.9G  505M  1.3G  29% /usr
tmpfs                    493M     0  493M   0% /dev/shm
tmpfs                    5.0M     0  5.0M   0% /run/lock
tmpfs                    493M     0  493M   0% /sys/fs/cgroup
/dev/mapper/vg0-var      1.4G   30M  1.3G   3% /var
/dev/mapper/vg0-home     465M  2.3M  434M   1% /home
/dev/sda1                464M   40M  396M  10% /boot
/dev/mapper/vg0-var_log  465M   30M  408M   7% /var/log
tmpfs                     99M     0   99M   0% /run/user/1000
```

```
  VG  #PV #LV #SN Attr   VSize  VFree
  vg0   1   6   0 wz--n- <9.52g 2.37g
```