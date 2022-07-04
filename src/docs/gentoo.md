# Gentoo Notes

These are notes covering Gentoo installation and configuration. Primarily to
have something documented for myself.

Preferred Gentoo configuration is:

- systemd
- no multilib
- gnome
- no split /usr
  - [ref](https://leo3418.github.io/2021/01/16/gentoo-merge-usr.html)

## Fedora Pre-Installation

Prior to installing Gentoo the latest Fedora should be installed and brought up
to date. Default installation options are fine as long as the installation
provides the following:

- btrfs root
  - Fedora installs to subvolume by default
- UEFI boot partition (ESP)
  - Mounted at `/boot/efi`
  - `/boot` is not a separate partition, though may be its own subvolume.
- GUI environment
- Working network
- Working mouse (bluetooth, etc)

Generally nothing special is required for this. Just click through the
installer.

### Disk setup

Current disk setup is two NVMe drives in RAID0. When installing Fedora the disk
setup steps should be checked to ensure the `RAID0` profile is used.

### Disk Layout

Remember, `/boot/efi` / `/efi` is the only other non-btrfs partition. The
following table shows the layout setup during a Fedora install.

| Name    | Mountpoint                  | Type              | Notes                                               |
| ------- | --------------------------- | ----------------- | --------------------------------------------------- |
| Fedora  | /                           | `btrfs` subvolume |                                                     |
| Gentoo  | /mnt/gentoo                 | `btrfs` subvolume |                                                     |
| home    | /home                       | `btrfs` subvolume |                                                     |
| boot    | /boot                       | `btrfs` subvolume |                                                     |
| ESP     | /boot/efi                   | EFI partition     | `/efi` on Gentoo                                    |
| Misc    | Anything                    | LVM misc space    | Useful for static backups                           |
| portage | /mnt/gentoo/var/tmp/portage | ZRAM              | Keeps portage workdir in memory rather than on disk |

The `Misc` space is leftover free space + (2 \* RAM size). Used for swap or
whatever else that would benefit from not being in a `btrfs` subvolume.

#### `fstab` Modifications

The `btrfs` mount options in the Fedora `/etc/fstab` should be updated to change
`compress` to `compress-force`. Just so `btrfs` compression is configured as desired from
the start.

#### ZRAM Portage WORKDIR

The ZRAM device needs to be manually set up using [`zram-generator`](https://github.com/systemd/zram-generator).

## Fedora Post-Installation

The default user account is not intended for sharing with Gentoo, at least until
Fedora better supports `systemd-homed`. Therefore the new user created on first
boot should be Fedora specific. The Fedora account is secondary to the Gentoo
account.

Ensure Fedora is brought up to date and that time is synced and correct. Additional
config may also be taken to make the environment more comfortable, such as setting up
Neovim and ZSH.

## Prepping the Gentoo Install

With a working Fedora environment the Gentoo installation can proceed. This will
mostly follow the [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64)
with deviations made for `systemd` usage. For example, right off the bat `systemd-nspawn`
should be used instead of `chroot` ([ref1](https://rich0gentoo.wordpress.com/2014/07/14/quick-systemd-nspawn-guide/), [ref2](https://github.com/cyber-duk/Gentoo-Systemd-Install-Guide)).

### Gentoo Mounts

Ensure the `ESP` has been mounted to `/mnt/gentoo/efi`. Aside from this everything else
should be ready to go provided everything has been set up as described in the
[Disk Layout](#disk-layout) table.

### Ebuild Repository

Ebuild repository should be set up _after_ the `stage3` has been extracted.

This is just to skip using `emerge-webrsync` for an initial Gentoo ebuild repository. As
`git` is not available in the `stage3` by default the Gentoo repository should be cloned
to `/mnt/gentoo/var/db/repos/gentoo` from Fedora. This way Portage can be configured to
sync over `git` from the start.

## Gentoo Installation

It is highly recommended to perform the Gentoo installation inside of `screen` so the
installation isn't interrupted by closing the terminal / SSH connection.

With the pre-installation environment ready to go the Gentoo install process may begin,
starting with the [Downloading the stage tarball](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Stage#Downloading_the_stage_tarball)
section.

### `stage3` Selection

Download and verify the current `stage3-amd64-nomultilib-systemd` tarball.

### `/usr` Merge

After the `stage3` has been extracted the steps to perform the `/usr` merge should be
taken. The instructions for performing this are here: [`/usr` merge steps](https://leo3418.github.io/2021/01/16/gentoo-merge-usr.html)

Follow the instructions for the `/usr` merge that configures it in the way `baselayout`
expects (the second form of the merge). Note that this involves a step before entering
the Gentoo environment and a step after entering the Gentoo environment.

### [Installing the Gentoo Base System](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Base)

Prior to doing any steps listed in the official handbook the Gentoo ebuild `git`
repository should be set up in `/mnt/gentoo/var/db/repos/gentoo` and the repository
`.conf` configured for `git` instead of `rsync`. The [GURU](https://wiki.gentoo.org/wiki/Project:GURU)
repository may also be added and configured.

#### `systemd-nspawn`

To allow login the `root` password in the Gentoo environment should be removed. This can
be accomplished via:

```bash
sed -i -e 's/^root:\*/root:/' /mnt/gentoo/etc/shadow
```

As mentioned previously `chroot` will not be used here. Instead `systemd-nspawn` will be
used so `systemd` can be active in the Gentoo environment. `chroot` setup may be ignored
and the Gentoo environment can be entered via:

```bash
systemd-nspawn -D /mnt/gentoo -j -b
```

### `systemd-boot`

<!-- TODO -->

## Gentoo Configuration

<!-- TODO -->

### Important Configuration

These files should likely be backup up in a git repo, sans things containing private keys.

- `/etc/portage`
- `/etc/containers`
  - for `podman`
- `/etc/dracut.conf.d`
  - for initramfs
- `/etc/efikeys`
  - for custom secureboot keys
- `/etc/fstab`
- `/etc/kernel`
  - config overrides
  - helper scripts (signing, etc)
- `/etc/secureboot`
  - copy of `efikeys`
- `/etc/systemd/zram-generator.conf`
  - For `/var/tmp/portage` ramdisk
  - Possibly swap too
- `/etc/polkit-1/rules.d/49-wheel.rules`
    - From https://leo3418.github.io/2021/04/24/gentoo-config-gnome-systemd.html#allow-users-in-wheel-group-to-use-their-own-credentials-for-authentication-in-gnome

`/var/lib/portage/world` contains the set of installed packages. Could be good to keep
this backed up as well.
