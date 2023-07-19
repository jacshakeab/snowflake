## :wrench: <samp>Installation</samp>

1. Download iso

   ```sh
   # Yoink nixos-unstable
   wget -O nixos.iso https://channels.nixos.org/nixos-unstable/latest-nixos-minimal-x86_64-linux.iso

   # Write it to a flash drive
   cp nixos.iso /dev/sdX
   ```

2. Boot into the installer.

3. Switch to root user: `sudo -i`

4. Partitioning

   We create a 512MB EFI boot partition (`/dev/nvme0n1p1`) and the rest will be our LUKS encrypted physical volume for LVM (`/dev/nvme0n1p2`).

   ```bash
   $ gdisk /dev/nvme0n1
   ```

   - `o` (create new empty partition table)
   - `n` (add partition, 512M, type ef00 EFI)
   - `n` (add partition, remaining space, type 8e00 Linux LVM)
   - `w` (write partition table and exit)

   Setup the encrypted LUKS partition and open it:

   ```bash
   $ cryptsetup luksFormat /dev/nvme0n1p2
   $ cryptsetup config /dev/nvme0n1p2 --label cryptroot
   $ cryptsetup luksOpen /dev/nvme0n1p2 enc
   ```

   We create two logical volumes, a 24GB swap parition and the rest will be our root filesystem

   ```bash
   $ pvcreate /dev/mapper/enc
   $ vgcreate vg /dev/mapper/enc
   $ lvcreate -L 24G -n swap vg
   $ lvcreate -l '100%FREE' -n root vg
   ```

   Format partitions

   ```bash
   $ mkfs.fat -F 32 -n boot /dev/nvme0n1p1
   $ mkswap -L swap /dev/vg/swap
   $ swapon /dev/vg/swap
   $ mkfs.btrfs -L root /dev/vg/root
   ```

   Mount partitions

   ```bash
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
   ```

5. Enable flakes

   ```bash
   $ nix-shell -p nixFlakes
   ```

6. Install nixos from flake

   ```bash
   $ nixos-install --flake 'github:jacshakeab/snowflake#carbon'
   ```

7. Reboot, login as root, and change the password for your user using passwd

8. Log in as your normal user.

9. Install the home manager configuration
   ```bash
   $ home-manager switch --flake 'github:jacshakeab/snowflake#jj@carbon'
   ```

<br>
<br>


## 💡 hardware-configuration.nix example

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
