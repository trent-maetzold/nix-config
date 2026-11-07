# nix-config

My machines and homes, declared in one flake, plus the reusable NixOS and
Home Manager modules they're built from. Nix installs every package. Configs I
edit often are out-of-store symlinks into this checkout, so I change them in
place, commit, push, and pull on the next machine.

This README is the spec: it describes the repo as it should be, and the code
follows it. Rules for changing it are in AGENTS.md.

It's also the backlog for every infrastructure project, tracked in
[beads](https://github.com/gastownhall/beads) (`bd`). Projects that grow
their own repo get their own beads there.

## Backlog

Each project is an epic, and its tasks sit under it with the dependencies
between them. `bd ready` lists what can start now; `bd show <id>` has the
context to pick a task up cold.

- Issues live in a Dolt database in the checkout's `.beads/`, shared by every
  worktree. Only its config (`.beads/config.yaml`, `metadata.json`,
  `.gitignore`) is committed.
- `bd dolt push` and `bd dolt pull` sync it through `refs/dolt/data` on
  GitHub. A fresh clone gets it with `bd bootstrap`.
- Priorities: P0 blocks everything else, P1 is next, P2 soon, P3 someday.

## Use the modules elsewhere

The public API is `modules.nixos.<name>` and `modules.homeManager.<name>`. A
module is independent of any user, machine or network. If it needs a value,
it takes it as a module arg with no default, and its first line names the
args it takes. Pass them as `specialArgs`/`extraSpecialArgs`:

```nix
{
  inputs.nix-config.url = "github:trent-maetzold/nix-config";

  outputs = {nix-config, home-manager, nixpkgs, ...}: {
    homeConfigurations."me@work" = home-manager.lib.homeManagerConfiguration {
      pkgs = nixpkgs.legacyPackages.x86_64-linux;
      extraSpecialArgs = {tailnet.loginServer = "https://hs.work.com";};
      modules = [
        nix-config.modules.homeManager.git
        nix-config.modules.homeManager.tmux
        ./home.nix
      ];
    };
  };
}
```

Variants are separate modules: if `git-home` and `git-work` exist, import the
one you want. `nixosModules`, `homeModules`, `nixosConfigurations` and
`homeConfigurations` are this flake's own machines and users, not API.

## Layout

```
flake.nix                         inputs, and the map of the infrastructure
modules/nixos/<domain>/<name>.nix one NixOS module: modules.nixos.<name>
modules/home/<domain>/<name>.nix  one Home Manager module: modules.homeManager.<name>
packages/<name>.nix               packages.<system>.<name>, for modules to use
lib/<name>.nix                    one shared helper, its contract on top (internal)
**/_lib/<name>.nix                one local helper, beside the file that calls it
hosts/<host>/default.nix          nixosConfigurations.<host>
hosts/<host>/*                    that host's hardware, disks, facter.json
users/<user>/default.nix          that user's homeModules, nixosModules, homeConfigurations
users/<user>/config/<app>/        that user's out-of-store configs, mirroring ~/.config
services/<service>/default.nix    nixosModules.<service>: a service this flake runs
services/<service>/*              that service's data files
checks/                           checks: the rules in this README, enforced
.beads/                           beads config; its database is local, synced via Dolt
secrets.yaml, .sops.yaml          every secret, and who can decrypt it
justfile                          operational recipes (install, deploy, rotate keys)
stacks/, terraform/               container stacks and OCI infra, still to be migrated
```

There are no profiles or bundles, no overlays and no global options. To see
what is on a machine or in a home, open its `hosts/` or `users/` file.
Everything it gets is listed in its imports.

### The map: `flake.nix`

`flake.nix` is the map of what exists. It holds:

- **Inputs**, grouped by purpose. Nix requires them to be a literal attrset
  in `flake.nix`.
- **Imports**: import-tree loads `modules/`, `lib/` and `checks/`. Every host,
  user and service directory is listed by hand, so the import list is the
  inventory.
- **`infra`**, a flake-parts module arg read only by host, user and service
  files. It
  holds the shared constants (domain, ACME email, time zone, tailnet login
  server, domain and subnet), every machine's addresses (tailnet, LAN, ssh),
  including machines this flake doesn't build, the Traefik dynamic config
  (`infra.routes`), and the Headscale extra DNS records (`infra.dns`). Each
  value is in the format its consumer takes. A helper builds them only if it
  removes real repetition, such as the boilerplate of many routes.

### Modules

- A module file sets exactly one `flake.modules.<class>.<name>` directly, with
  no wrapping `let`. Its name is the file name, unique within its class. Data
  files a module reads sit in a directory beside it (`<name>/`).
- The directory name under `modules/<class>/` is the domain:
  - `nix`: Nix itself.
  - `system`: host OS behaviour.
  - `network`: tailnet and routing on the client side.
  - `services`: servers that clients connect to (headscale, traefik, honcho).
  - `shell`: the terminal.
  - `dev`: writing code.
  - `ai`: agents, models and memory.
  - `desktop`: graphical sessions.
  - `infra`: running fleets.
- Importing a module enables it, so modules have no `enable` options.
  Modules never import each other; the host or user lists every one it uses.
- A module uses this flake's inputs and `packages` through its own closure
  (`inputs.<x>`, `self.packages.<system>.<name>`), so a consumer needs no
  overlay.
- `packages/` holds plain derivations (`<name>.nix` or `<name>/package.nix`).
  `lib.filesystem.packagesFromDirectoryRecursive` turns them into the
  `packages` output. That is the only `callPackage` in the repo; modules,
  hosts, users and services never call it.
- Modules never declare secrets. A secret a module needs is an arg holding the
  path of a file containing only its value. Where an upstream option already
  takes the file (`services.tailscale.authKeyFile`), the host sets that option
  and the module takes no arg.

### Helpers

Reach for upstream first: nixpkgs `lib`, `sops.templates`, `pkgs.writers`,
or a mature flake. When a computation still reads as noise in the file that
needs it, it becomes a helper in its own file, so the caller reads as config:

- **Local** by default: `_lib/<name>.nix` beside the one file that calls it.
  It's a plain function, imported by relative path (import-tree skips `_`
  paths).
- **Shared** when more than one file calls it, or it's general-purpose (for
  example, merging an attrset with a JSON file): `lib/<name>.nix` sets
  `_module.args.<name>`, and flake-parts files take it as an arg
  (`{mergeJson, ...}:`).
- The name says exactly what it returns and from what, so the call reads
  without opening the helper.
- The file opens with its contract: every arg and what it means, what it
  returns, and one example call.

Helpers are internal: there is no `lib` output. The flake exports modules, plus
the `packages` those modules use.

### Services

A service splits in two:

- **The reusable part** is a module in `modules/nixos/services/`. Like every
  module, it knows nothing about where it runs.
- **The instance** is `services/<service>/default.nix`, which sets
  `flake.nixosModules.<service>`. It imports the module, then adds what
  belongs to this deployment: values from `infra`, bind addresses, its
  `sops.secrets`, persistence, and data files beside it. For example,
  `services/traefik` sets `services.traefik.dynamicConfigOptions = infra.routes`.

The host that runs a service imports its `nixosModules.<service>`, so the host
file still lists everything on the machine. Services and users share the
`nixosModules` namespace, so their names are unique.

### Hosts

`hosts/<host>/default.nix` sets `flake.nixosConfigurations.<host>` with
`nixpkgs.lib.nixosSystem`. It contains:

- `specialArgs`, built from `infra`.
- Every module the host uses, including `modules.nixos.home-manager`, which
  loads Home Manager into NixOS once per host.
- The `nixosModules.<user>` of each person who logs in, and the
  `nixosModules.<service>` of each service it runs.
- The host's own config: hardware, disks, `stateVersion`, its
  `sops.secrets`, and anything that names this machine.

A machine that runs another, such as a VM, declares it in its own file.

### Users and configs

`users/<user>/default.nix` builds everything for one person:

- `flake.homeModules.<user>`, plus variants like `<user>-server` when machines
  need different homes. Each lists its home modules and contains the user's
  personal settings and their `stateVersion`.
- `flake.nixosModules.<user>`, plus the same variants. Each creates the
  account (`users.users.<user>`) and sets `home-manager.users.<user>` to the
  matching home module and `home-manager.extraSpecialArgs` from `infra`.
  NixOS hosts run Home Manager as this NixOS module, never standalone.
- `flake.homeConfigurations."<user>@<host>"` for each machine that isn't
  NixOS (Omarchy), with `targets.genericLinux.enable`.

How often a config changes decides how it is managed:

- **Changes often** (harness, editor, MCP, skills, agent rules): plain files
  in `users/<user>/config/<app>/`, linked with
  `config.lib.file.mkOutOfStoreSymlink "<checkout>/users/<user>/config/<app>"`.
  Edits take effect immediately; commit them like code.
- **Set once** (git identity, shell, tmux): Home Manager options, read-only in
  the store.

On Omarchy, Omarchy owns its desktop configs (Hyprland, Waybar and the rest)
until one is adopted. To adopt one (nvim and herdr, for example):

1. Move Omarchy's config into the repo:
   `mv ~/.config/<app> users/<user>/config/<app>`.
2. Add the link, then switch.

From then on, the repo is the only copy. Home Manager refuses to overwrite a
file it doesn't manage, so a switch that would clobber an unadopted config
fails until that config is moved. Nothing here sets `backupFileExtension`,
`-b` or `force`. An app that hasn't been adopted is left to Omarchy.

Nix owns every installed package; there's no mise and no global npm or pip.
A project's own toolchain lives in that project's flake devShell, loaded by
direnv.

### Checkouts and worktrees

I own my checkouts in `~/Projects` and `~/Work`. This repo's live checkout is
`~/Work/nix-config`. Agents own `~/.worktrees` and edit code only there:

- **By hand:** `~/.worktrees/<repo>/<branch>`. `<repo>` is the repository
  name from `origin` (`nix-config`), or the directory name without one.
- **By a tool:** each tool that creates worktrees owns
  `~/.worktrees/.<tool>/`, and the layout inside is the tool's own. The dot
  keeps the tool directories from colliding with a repo name.
  - herdr: `.herdr/<repo>/<branch>`, via `[worktrees] directory`.
  - Claude Code: `.claude/<repo>/<name>`, via `WorktreeCreate`/`WorktreeRemove`
    hooks.
  - cursor-agent: `.cursor/<repo>/<name>`, via `CURSOR_WORKTREES_ROOT`.
  - Codex: `.codex/<bucket>/<repo>`, via `[desktop] git-worktree-root`.
  - Paseo: `.paseo/<hash>/<slug>`, via `worktrees.root`.

The global agent rules are one file, `~/.agents/AGENTS.md`, linked into every
harness that reads a global file (Claude Code, Codex, pi, opencode). It
carries the worktree rule. These configs live in `users/<user>/config/`.

This repo's live checkout is the running config, because the out-of-store
links point into it. Its branch is switched only to pull, never to review.
To review, diff against the branch instead.

## Machines

| Machine   | What                                             | Built as              | Users                             |
| --------- | ------------------------------------------------ | --------------------- | --------------------------------- |
| muninn    | Omarchy workstation                              | `homeConfigurations`  | trent                             |
| frigg     | family PC (Cinnamon, GTX 980)                    | `nixosConfigurations` | trent, justine, leo, violet, june |
| thor      | gaming PC, headless for now                      | `nixosConfigurations` | trent                             |
| heimdall  | public edge on OCI (Traefik, Headscale), aarch64 | `nixosConfigurations` | trent (server)                    |
| yggdrasil | home server (agents, models, VMs)                | `nixosConfigurations` | trent (server)                    |
| mimir     | Unraid VM on yggdrasil, HBA passed through       | `infra` only          | -                                 |

## Secrets

[sops-nix](https://github.com/Mic92/sops-nix). Every secret is in the root
`secrets.yaml`, the single source of truth, under a group named for what uses
it (`tailscale/ts-authkey`) or under `shared/`. The host or user file that
needs a secret declares `sops.secrets.<n>.key` and passes the path on.
`.sops.yaml` encrypts the file to the age master key (Bitwarden item
`age-master`), every host's `/var/lib/age/key.txt`, and the user key of every
standalone home.

```sh
rbw unlock && sops edit secrets.yaml   # edit
sops updatekeys secrets.yaml           # after changing a recipient
```

## Commands

```sh
nix fmt                                            # format (alejandra)
nix flake check --no-build                         # evaluate every config and check
home-manager switch --flake .#trent@muninn         # a non-NixOS home
sudo nixos-rebuild switch --flake .#<host>         # on the host
nixos-rebuild switch --flake .#<host> --target-host <host> --build-host <host> --sudo
```

Servers are deployed over ssh with the last command until GitOps replaces it.
There's no colmena.
