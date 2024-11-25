---
date: '2024-06-24T23:52:31-05:00'
draft: false
title: 'NixOS inside LXC (without flakes) made easy'
categories: ["tech"]
---

LXCs are great, but LXC templates that are booted and instantly accessible over SSH and running services are better. The best part is that it's done without iterative, stateful tools like Ansible or needing to provision before running a deployment tool like Colmena. It just works. 

lxc.nix is where all the nitty-gritty is. A sample here is provided, and any other nix files need to be imported here to be baked into the tarball. 

```nix
{ modulesPath, ... }:
{
  imports = [
    ./base.nix
    (modulesPath + "/virtualisation/proxmox-lxc.nix")
  ];
  boot.isContainer = true;
  # Supress systemd units that don't work because of LXC
  systemd.suppressedSystemUnits = [
    "dev-mqueue.mount"
    "sys-kernel-debug.mount"
    "sys-fs-fuse-connections.mount"
  ];
}
```

base.nix will be loaded in upon boot. Note that this is Nix, and if you update configuration.nix on the host later, this will be overwritten. Treat each template as stateful; make edits and recompile the tarball as needed.

```nix
{ pkgs, ...}: 
{
  system.stateVersion = "24.11";
  users.users.nixos =
    {
      isNormalUser = true;
      extraGroups = [ "wheel" ];
      openssh.authorizedKeys.keys = [
        "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIbtvGsRqXeaWMxilWAfzqCi2ng6Aiiy7xp8oBh+Ugiq nate@repono"
      ];
    };
  services.openssh = {
    enable = true;
    settings.PasswordAuthentication = false;
    settings.KbdInteractiveAuthentication = false;
    settings.PermitRootLogin = "no";
  };
  security.sudo.wheelNeedsPassword = false;
  environment.systemPackages = with pkgs; [
    vim
    binutils
    wget 
  ];
}
```

The secret is in https://github.com/nix-community/nixos-generators. Build using

```console
nix run github:nix-community/nixos-generators -- --format proxmox-lxc --configuration lxc.nix
```

In PVE, upload the template in Datacenter > pve > local (pve). 


When creating the container, make sure Nesting is checked. 

![Nesting](/img/nesting-checked.png)
To avoid the [busctl](https://github.com/nix-community/nixos-generators/issues/319) error [patched 07/24](https://github.com/NixOS/nixpkgs/pull/328682)

1. Build image
2. Run image
3. Update nix channels as root
4. Reboot.
5. Install desired configuration.nix and attempt to switch to it
6. Force a reboot sudo reboot -f





