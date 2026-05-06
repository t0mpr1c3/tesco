# Instructions for setting up Claude Code on WSL2

1. First, you'll need to install WSL2. You'll need to open an IT ticket for permissions on a TESCO machine.

```PowerShell
wsl --install --no-distribution
```

2.  Next, install the linux environment that we'll be using to create the claude VM(s).
    We will be using NixOS. It's more complicated than a normal linux install, but you better control over the process.
    Follow steps 1-7 from https://www.greghilston.com/post/nixos-on-wsl/#installing-wsl2-with-nixos
