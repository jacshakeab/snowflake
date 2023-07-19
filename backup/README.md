:wrench: Installation

Download iso

# Yoink nixos-unstable
wget -O nixos.iso https://channels.nixos.org/nixos-unstable/latest-nixos-minimal-x86_64-linux.iso

# Write it to a flash drive
cp nixos.iso /dev/sdX
Boot into the installer.

Switch to root user: sudo -i

Partitioning

We create a 512MB EFI boot partition (/dev/nvme0n1p1) and the rest will be our LUKS encrypted physical volume for LVM (/dev/nvme0n1p2).

$ gdisk /dev/nvme0n1
o (create new empty partition table)
n (add partition, 512M, type ef00 EFI)
n (add partition, remaining space, type 8e00 Linux LVM)
w (write partition table and exit)
Setup the encrypted LUKS partition and open it:

$ cryptsetup luksFormat /dev/nvme0n1p2
$ cryptsetup config /dev/nvme0n1p2 --label cryptroot
$ cryptsetup luksOpen /dev/nvme0n1p2 enc
We create two logical volumes, a 24GB swap parition and the rest will be our root filesystem

$ pvcreate /dev/mapper/enc
$ vgcreate vg /dev/mapper/enc
$ lvcreate -L 24G -n swap vg
$ lvcreate -l '100%FREE' -n root vg
Format partitions

$ mkfs.fat -F 32 -n boot /dev/nvme0n1p1
$ mkswap -L swap /dev/vg/swap
$ swapon /dev/vg/swap
$ mkfs.btrfs -L root /dev/vg/root
Mount partitions

$ mount -t btrfs /dev/vg/root /mnt

# Create the subvolumes
$ btrfs subvolume create /mnt/root
$ btrfs subvolume create /mnt/home
$ btrfs subvolume create /mnt/nix
$ btrfs subvolume create /mnt/log
$ umount /mnt

# Mount the directories
$ mount -o subvol=root,compress=zstd,noatime,ssd,space_cache=v2 /dev/vg/root /mnt
$ mkdir -p /mnt/{home,nix,var/log}
$ mount -o subvol=home,compress=zstd,noatime,ssd,space_cache=v2 /dev/vg/root /mnt/home
$ mount -o subvol=nix,compress=zstd,noatime,ssd,space_cache=v2 /dev/vg/root /mnt/nix
$ mount -o subvol=log,compress=zstd,noatime,ssd,space_cache=v2 /dev/vg/root /mnt/var/log

# Mount boot partition
$ mkdir /mnt/boot
$ mount /dev/nvme0n1p1 /mnt/boot
Enable flakes

$ nix-shell -p nixFlakes
Install nixos from flake

$ nixos-install --flake 'github:jacshakeab/snowflake#captainamerica'
Reboot, login as root, and change the password for your user using passwd

Log in as your normal user.

Install the home manager configuration

$ home-manager switch --flake 'github:jacshaceab/snowflake#jj@captainamerica'




# hardware-configuration.nix example

```
{
  config,
  lib,
  pkgs,
  modulesPath,
  ...
}: {
  imports = [(modulesPath + "/installer/scan/not-detected.nix")];

  boot = {
    initrd.luks.devices.luksroot = {
      device = "/dev/disk/by-label/cryptroot";
      preLVM = true;
      allowDiscards = true;
    };

    initrd.availableKernelModules =
      [
        "xhci_pci"
        "thunderbolt"
        "nvme"
        "usb_storage"
        "sd_mod"
      ]
      ++ config.boot.initrd.luks.cryptoModules;

    initrd.kernelModules = ["dm-snapshot"];
    kernelModules = ["kvm-intel"];
    extraModulePackages = [];
  };

  fileSystems = {
    "/" = {
      device = "/dev/disk/by-label/root";
      fsType = "btrfs";
      options = [
        "subvol=root"
        "compress=zstd"
        "noatime"
        "ssd"
        "space_cache=v2"
      ];
    };

    "/home" = {
      device = "/dev/disk/by-label/root";
      fsType = "btrfs";
      options = [
        "subvol=home"
        "compress=zstd"
        "noatime"
        "ssd"
        "space_cache=v2"
      ];
    };

    "/nix" = {
      device = "/dev/disk/by-label/root";
      fsType = "btrfs";
      options = [
        "subvol=nix"
        "compress=zstd"
        "noatime"
        "ssd"
        "space_cache=v2"
      ];
    };

    "/var/log" = {
      device = "/dev/disk/by-label/root";
      fsType = "btrfs";
      options = [
        "subvol=log"
        "compress=zstd"
        "noatime"
        "ssd"
        "space_cache=v2"
      ];
      neededForBoot = true;
    };

    ${config.boot.loader.efi.efiSysMountPoint} = {
      device = "/dev/disk/by-label/boot";
      fsType = "vfat";
    };
  };

  swapDevices = [
    {device = "/dev/disk/by-label/swap";}
  ];

  # Enables DHCP on each ethernet and wireless interface. In case of scripted networking
  # (the default) this is the recommended approach. When using systemd-networkd it's
  # still possible to use this option, but it's recommended to use it in conjunction
  # with explicit per-interface declarations with `networking.interfaces.<interface>.useDHCP`.
  networking.useDHCP = lib.mkDefault true;
  # networking.interfaces.wlp0s20f3.useDHCP = lib.mkDefault true;

  nixpkgs.hostPlatform = lib.mkDefault "x86_64-linux";
  powerManagement.cpuFreqGovernor = lib.mkDefault "powersave";
  hardware.cpu.intel.updateMicrocode = lib.mkDefault config.hardware.enableRedistributableFirmware;
}
``` 
