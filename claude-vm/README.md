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

```PowerShell
whoami
```

5.  
