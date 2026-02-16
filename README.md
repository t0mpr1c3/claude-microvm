# claude-vm

Run [Claude Code](https://docs.anthropic.com/en/docs/claude-code) in an isolated NixOS microVM via [microvm.nix](https://github.com/microvm-nix/microvm.nix) (QEMU+KVM). Your project directory is mounted read-write at `/work` inside the guest via virtiofs — no root required.

## Prerequisites

- [Nix](https://nixos.org/) with flakes enabled
- KVM support (`/dev/kvm`)

## Quick start

```sh
# Build and run with current directory mounted at /work
make vm.run

# Mount a specific project directory
WORK_DIR=/path/to/project make vm.run
```

You'll be auto-logged in as `claude` and dropped into `/work`. Run `claude` to start a session.

## Usage from anywhere

### `nix run` (no install)

```sh
# From the repo directory
WORK_DIR=. nix run

# From a local checkout
WORK_DIR=/path/to/project nix run /path/to/this/repo

# Directly from git
WORK_DIR=. nix run github:yourorg/claude-vm
```

### Install to PATH

```sh
nix profile install github:yourorg/claude-vm

# Now available everywhere
WORK_DIR=/path/to/project microvm-run
```

### As a flake input

Add as a dependency in another project's `flake.nix`:

```nix
{
  inputs.claude-vm.url = "github:yourorg/claude-vm";

  outputs = { nixpkgs, claude-vm, ... }:
    let system = "x86_64-linux"; in {
      devShells.${system}.default = nixpkgs.legacyPackages.${system}.mkShell {
        packages = [ claude-vm.packages.${system}.vm ];
      };
    };
}
```

Then `nix develop` gives you `microvm-run` in the shell.

## How it works

### virtiofs (host directory sharing)

The host `WORK_DIR` is shared into the VM at `/work` using virtiofs. A `virtiofsd` daemon is started automatically as a systemd user service (`claude-vm-virtiofsd`) — no root or sudo needed. It runs unprivileged in a user namespace with UID/GID translation so files created inside the VM are owned by your host user.

The daemon persists between VM restarts for fast re-launches. It automatically restarts when `WORK_DIR` changes. Manage it with:

```sh
systemctl --user status claude-vm-virtiofsd
systemctl --user stop claude-vm-virtiofsd
```

### Shutting down

Exiting the shell runs `sudo poweroff` automatically. You can also run it manually.

## Customization

### Exposing ports

Port 7160 (Claude OAuth callback) is forwarded by default. To expose additional ports, edit `flake.nix`:

```nix
microvm.qemu.extraArgs = [
  "-netdev" "user,id=usernet,hostfwd=tcp::7160-:7160,hostfwd=tcp::8080-:8080"
  "-device" "virtio-net-device,netdev=usernet"
];
networking.firewall.allowedTCPPorts = [ 7160 8080 ];
```

Rebuild with `make vm`.

### VM specs

| Resource | Default |
|----------|---------|
| RAM      | 4096 MB |
| vCPUs    | 4       |
| Network  | User-mode (SLiRP) |
| Store    | Host nix store via 9p (read-only) |
| Work dir | Host directory via virtiofs (read-write) |
