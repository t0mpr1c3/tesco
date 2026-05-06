# Instructions for setting up Claude Code on WSL2


## Environment

Let's get set up with Claude in WSL locally and try to run VScode in it. If that doesn't work well, we can try VMware. 
 
The basic idea is that we are going to install a Linux VM in WSL, and from there spin up a secure sandbox (called a microVM) 
every time we want to run Claude. Each instance of Claude runs from inside its own sandbox. By default the sandboxes have 
restrictive permissions so that Claude only has access to a specified work directory. These restrictive permissions, 
together with the fact the microVM cannot directly access the system commands in the parent OS, are what provides some
security when you let Claude loose on your computer with the ability to write its own shell scripts and run them.
 
First, you'll need to install WSL2. You'll need to open an IT ticket for permissions on a TESCO machine. Then follow the 
instructions below.
 
When you have got the NixOS VM properly set up, Claude Code should run automatically in the console when you start up an 
instance using one of the methods described in https://github.com/t0mpr1c3/claude-microvm/blob/main/README.md 

Authenticate using method 1 (by logging in via a URL). I will set my email to automatically forward emails from Anthropic to you. 

To change Claude's settings, edit `~/.claude/settings.json` in the NixOS VM.
 
The point of the microVMs is that they are lightweight sandboxes to run Claude instances in. By default, Claude only has access 
to `$WORK_DIR` and its subdirectories, and requires explicit authorization for external access through any kind of port. You can 
of course override these defaults. This should be done by whitelisting. That means, you should only allow specific exceptions to
the defaults, whether or not you run Claude with `--dangerously-skip-permissions` (also known as "YOLO mode").
 
**DO NOT TRUST CLAUDE**. I recommend that all external access including to your file system is read-only. This may slow 
down your workflow a tiny bit, but it will stop Claude from bricking your computer and deleting the repo.
 

## Instructions

1. First, you'll need to install WSL2. You'll need to open an IT ticket for permissions on a TESCO machine.
   Your machine must have virtualization enabled. If you have VMware installed, it already does.
   Open PowerShell with admin privileges and type:

```PowerShell
wsl --install --no-distribution
```

2.  Next, install the linux environment that we'll be using to create the claude VM(s).
    We will be installing NixOS. 
    It's more complicated than a normal Linux install, but you get better control over the process.
    Download the latest `nixos.wsl` to your Downloads folder from the
    [NixOS-WSL releases page](https://github.com/nix-community/NixOS-WSL/releases/latest).
    In PowerShell, type :

```PowerShell
cd $env:USERPROFILE\Downloads
wsl --import nixos $env:USERPROFILE\NixOS\ nixos.wsl
wsl -s nixos
```

3.  NixOS should now be set as the default WSL environment. To enter the environment, go to your Powershell
    window and type:

```PowerShell
wsl -u root
```

4.  You should see a welcome message and a different console prompt. Check that you are logged in as `root`:

```sh
whoami
```

5.  Next we need to edit a configuration file:

```sh
nano /etc/nixos/configuration.nix
```

6.  Hold down Ctrl+K until the file is empty. Then paste in the following text, and Ctrl+X to save it:

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
    gnumake
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

7. Update the environment:

```sh
nixos-rebuild boot
```

8. Set the password for account `tesco`:

```sh
passwd tesco
```

9. Exit to PowerShell:

```bash
exit
```

10. Terminate the NixOS VM, and then restart it. After this, you should be logged in as user `tesco`:

```PowerShell
wsl -t nixos
wsl
```

11.  Now we need to edit another file:

```sh
sudo nano /etc/nixos/flake.nix
```

12. Enter and save the following text:

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
  };

  outputs = {
    self,
    nixpkgs,
    nixpkgs-unstable,
    claude-vm
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
      ];
      system.stateVersion = "25.11";
    };

    # `nix develop /etc/nixos` starts a shell that provides `microvm-run`
    devShells.${system}.default = nixpkgs-unstable.legacyPackages.${system}.mkShell {
      packages = [ claude-vm.packages.${system}.vm ];
    };
  };
}
```

13. Rebuild the environment:

```sh
sudo nixos-rebuild switch --impure
```

14. Make files for Claude settings and secrets:

```sh
mkdir -p ~/.claude
echo '{}' > ~/.claude/settings.json
```

14. Clone the directory that sets up the microVM(s) that Claude runs in and view the README that contains further instructions:

```sh
cd ~
git clone https://github.com/t0mpr1c3/claude-microvm.git
cat claude-microvm/README
```

## Running Claude from VScode

I don't know anything about running Claude Code with VScode but to sandbox it from WSL you would need to do the following steps.

1. Configure WSL to use wayland. Recent installs should run without any configuration.

2. Install VScode in NixOS: e.g. edit `/etc/nixos/configuration.nix` to add `vscode` to `environment.systemPackage`s, then do `sudo nixos-rebuild switch --impure`.

3. Add whatever plugins you need, e.g. https://marketplace.visualstudio.com/items?itemName=TheQtCompany.qt, https://marketplace.visualstudio.com/items?itemName=anthropic.claude-code.

4. Run VScode inside the microVM.
   * Easy way: press Ctrl-C after the microVM has started up but before Claude Code has loaded, then run VScode from the command line.
   * Smart way: edit `programs.bash.interactiveShellInit` in `~/.claude-microvm/flake.nix` so that it runs VSCode instead of Claude.
     
You can use VScode with the WSL plugin to edit files in your NixOS VM, but don't try to use this set up together with the Claude plugin. 
What provides security is not the VM itself but the microVM sandboxes that run inside it.

