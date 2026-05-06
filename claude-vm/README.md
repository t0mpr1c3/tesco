# Instructions for setting up Claude Code on WSL2

1. First, you'll need to install WSL2. You'll need to open an IT ticket for permissions on a TESCO machine.
   Your machine must have virtualization enabled. If you have VMware installed, it already does.
   Open PowerShell with admin privileges and type

```PowerShell
wsl --install --no-distribution
```

2.  Next, install the linux environment that we'll be using to create the claude VM(s).
    We will be installing NixOS. 
    It's more complicated than a normal Linux install, but you get better control over the process.
    Download the latest nixos.wsl to your Downloads folder from the [NixOS-WSL releases page](https://github.com/nix-community/NixOS-WSL/releases/latest).
    In PowerShell, type 

```PowerShell
cd $env:USERPROFILE\Downloads
wsl --import nixos $env:USERPROFILE\NixOS\ nixos.wsl
wsl -s nixos
```

3.  NixOS should now be set as the default WSL environment. To enter the environment, go to your Powershell
    window and type

```PowerShell
wsl -u root
```

4.  You should see a welcome message and a different console prompt. To check that you are logged in as `root`, type

```bash
whoami
```

5.  Next we need to edit a configuration file. Type

```bash
nano /etc/nixos/configuration.nix
```

6.  Hold down Ctrl+K until the file is empty. Then paste in the following text, and Ctrl+X to save it.

```nix
# Edit this configuration file to define what should be installed on
# your system. Help is available in the configuration.nix(5) man page, on
# https://search.nixos.org/options and in the NixOS manual (`nixos-help`).

# NixOS-WSL specific options are documented on the NixOS-WSL repository:
# https://github.com/nix-community/NixOS-WSL

{ config, lib, pkgs, ... }:

{
  imports = [
    # include NixOS-WSL modules
    <nixos-wsl/modules>
  ];

  # Define a user account. Don't forget to set a password with ‘passwd’.
  users.users.tesco = {
    isNormalUser = true;
    extraGroups = [ "wheel" ]; # Enable ‘sudo’ for the user.
  };

  wsl.enable = true;
  wsl.defaultUser = "tesco";
  networking.hostName = "nixos";

  # List packages installed in system profile.
  # You can use https://search.nixos.org/ to find more packages (and options).
  environment.systemPackages = with pkgs; [
    # Flakes clones its dependencies through the git command,
    # so git must be installed first
    git
    vim # The Nano editor is also installed by default.
    wget
    inetutils
    tree
  ];

  programs = {
    git.enable = true;
  };

  # Enable the Flakes feature and the accompanying new nix command-line tool
  nix.settings = {
    experimental-features = [ "nix-command" "flakes" ];
    trusted-users = [ "root" "tesco" ];
  };

  # Set the default editor to vim
  environment.variables.EDITOR = "vim";

  # This value determines the NixOS release from which the default
  # settings for stateful data, like file locations and database versions
  # on your system were taken. It's perfectly fine and recommended to leave
  # this value at the release version of the first install of this system.
  # Before changing this value read the documentation for this option
  # (e.g. man configuration.nix or on https://nixos.org/nixos/options.html).
  system.stateVersion = "25.11"; # Did you read the comment?
}
```

7. Update the environment by typing

```bash
nixos-rebuild boot
```

8. Set the password for account `tesco`.

```bash
passwd tesco
```

9. Exit to PowerShell by typing

```bash
exit
```

10. Terminate the NixOS VM, and then restart it. After this, you should be logged in as user `tesco`.

```PowerShell
wsl -t nixos
wsl
```

11.  Now we need to edit another file. Type

```bash
sudo nano /etc/nixos/flake.nix
```

12. Enter and save the following text

```nix
{
  description = "Set up microvm environment";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.11";
    nixpkgs-unstable.url = "github:nixos/nixpkgs/nixos-unstable";
    claude-vm = {
      url = "github:t0mpr1c3/claude-microvm";
      inputs.nixpkgs.follows = "nixpkgs";
    };
    sops-nix = {
      url = "github:Mic92/sops-nix";
      inputs.nixpkgs.follows = "nixpkgs";
    };
    home-manager = {
      url = "github:nix-community/home-manager/release-25.11";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = {
    self,
    nixpkgs,
    nixpkgs-unstable,
    claude-vm,
    sops-nix,
    home-manager
  }:
    let
      system = "x86_64-linux";
    in
  {
    nixosConfigurations.nixos = nixpkgs.lib.nixosSystem {
      modules = [
        ./configuration.nix
        ({ pkgs, config, lib, ... }:
        {
          nixpkgs.hostPlatform = lib.mkDefault "x86_64-linux";
        })
        sops-nix.nixosModules.sops
      ];
      system.stateVersion = "25.11";
    };

    homeConfigurations.tesco = home-manager.lib.homeManagerConfiguration {
      pkgs = nixpkgs.legacyPackages.x86_64-linux;
      modules = [ ./home.nix ];
    };

    # `nix develop /etc/nixos` starts a shell that provides `microvm-run`
    devShells.${system}.default = nixpkgs-unstable.legacyPackages.${system}.mkShell {
      packages = [ claude-vm.packages.${system}.vm ];
    };
  };
}
```

