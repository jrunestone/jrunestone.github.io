+++
title = "How to install Incus on NixOS"
template = "post.html"
date = 2025-09-13
path = "/how-to-install-incus-lxd-on-nixos"
+++

Set up Incus (LXD) container and VM manager on NixOS with a bridged network configuration as an alternative to Proxmox.
Instances will be visible on the network if you start them with `--profile bridged`.

<!-- toc -->

## Virtualisation config
This config disables the Incus firewall and uses NixOS firewall instead. You'll see both configurations being used elsewhere.
If you want to use zfs check out the links section.

```nix
{
  virtualisation.incus = {
    enable = true;
    ui.enable = true;
    package = pkgs.incus; # use incus-lts for lts

    preseed = {
      networks = [
        {
          name = "internalbr0";
          type = "bridge";
          description = "Internal/NATted bridge";

          config = {
            "ipv4.address" = "auto";
            "ipv4.nat" = "true";
            "ipv4.firewall" = "false";
            "ipv6.address" = "auto";
            "ipv6.nat" = "true";
            "ipv6.firewall" = "false";
          };
        }
      ];

      profiles = [
        {
          name = "default";
          description = "Default Incus Profile";
        
          devices = {
            eth0 = {
              name = "eth0";
              network = "internalbr0";
              type = "nic";
            };

            root = {
              path = "/";
              pool = "default";
              type = "disk";
            };
          };
        }

        {
          name = "bridged";
          description = "Instances bridged to LAN";
          
          devices = {
            eth0 = {
              name = "eth0";
              nictype = "bridged";
              parent = "externalbr0";
              type = "nic";
            };
            
            root = {
              path = "/";
              pool = "default";
              type = "disk";
            };
          };
        }
      ];

      storage_pools = [
        {
          config = {
            source = "/var/lib/incus/storage-pools/default";
          };

          driver = "dir";
          name = "default";
        }
      ];
    };
  };
}
```

## Networking config
Make sure to replace the relevant fields.

```nix
{
  networking = {
    nftables.enable = true;
    useDHCP = false;
    tempAddresses = "disabled";
    hostId = "cf9fe3d2"; # change this to something unique on your network
    hostName = "jr-homelab"; # change this
    firewall.trustedInterfaces = ["internalbr0"];
    
    bridges = {
      externalbr0 = {
        interfaces = ["enp1s0"]; # change this to your network adapter
      };
    };

    interfaces = {
      externalbr0 = {
        useDHCP = true;
        macAddress = "a6:3f:8a:0e:bf:19"; # change this, this is just a randomly generated mac
      };
    };
  };
}
```

## Users config
Make sure to use your own username.

```nix
{
  users.users.jr.extraGroups = ["incus-admin"];
}
```

## OpenSSH config
If you're installing Incus on a dedicated server you want to be able to access it via SSH.
Make sure to replace the relevant fields.

```nix
{
  services.openssh = {
    enable = true;
    ports = [22];

    settings = {
      AllowUsers = ["jr"];
      PasswordAuthentication = false;
      KbdInteractiveAuthentication = false;
      PermitRootLogin = "no";
    };
  };

  users.users.jr.openssh.authorizedKeys.keys = [
    "your-public-key-here"
  ];

  networking.firewall.allowedTCPPorts = [22];
}
```

## Test
After building, you can test your installation:

```bash
ssh jr@homelab
incus launch images:debian/trixie test-container --profile bridged
incus ls
incus console --show-log test-container
incus shell test-container
incus rm test-container -f
```

You can log in to the web based UI by browsing to your Incus server's IP.
You then first need to set the https port: 

```bash
incus config set core.https_address :8443
```

Then browse to `https://your-incus-server-ip:8443`.

## Links & references
[Incus on NixOS Wiki](https://wiki.nixos.org/wiki/Incus)<br>
[Install Incus and ZFS on NixOS](https://blog.hetherington.uk/2025/01/setting-up-incus-with-zfs-on-nixos/)
