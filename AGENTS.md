# AGENTS.md

nix-config: my machines and homes, plus reusable NixOS and Home Manager
modules. README.md is the spec. It describes the repo as it should be, and it
comes before code.

## Workflow

Work only in a worktree at `~/.worktrees/<repo>/<branch>`, never in the live
checkout. The live checkout is the running config: the out-of-store links
point into it, so an edit or branch switch there changes the live system.

Every change starts from a bead (README's Backlog). Run `bd ready`, then
`bd update <id> --claim` before you start, and name the bead in the branch and
PR (`nc-a3f-short-name`). Add notes to it as you go, so the next session can
resume. Close it with `bd close <id> --reason "..."` once the PR merges, then
`bd dolt push`. Work you discover gets its own bead
(`--deps discovered-from:<id>`), not a TODO in code. Never create or rewrite
epics without asking; tasks under an agreed epic are fine.

Every change goes in this order:

1. **Spec.** If the change alters what the repo looks like (layout, a rule, a
   machine, a user, a service, the public API), edit README.md first. If a request
   contradicts README.md, say so and settle it before touching code.
2. **Checks.** Make `checks/` express the new spec, so they fail on the
   current code and will pass on the change.
3. **Code.** Change code until the checks pass.
4. **Verify.** Run `nix fmt`, then `nix flake check --no-build`. Report
   failures as they are.

Where the code and README.md disagree, README.md is right and the code is the
work left. Never edit the spec to match the code. If you're asked to ("just
update the README to match"), push back. Name the rule the code breaks and
ask whether the design is really changing. If it is, the spec changes as a
design decision, on its merits, and the code then follows it. It never
changes because the code drifted.

## Checks

`checks/` holds flake-parts modules that set `perSystem.checks`:

- They evaluate every `nixosConfigurations.<host>` and every
  `homeConfigurations.<home>`.
- They enforce the README's structural rules. Examples: a module file sets
  exactly one `flake.modules.<class>.<name>`, and its class matches
  `modules/nixos` or `modules/home`. No module mentions a host, user, domain
  or address from `infra`. Every host, user and service directory is imported
  in `flake.nix`. `nixosModules` names are unique across users and services.
  Every helper file opens with its contract. The flake has no `lib` output.
- They inspect evaluated config, not source, so values set indirectly or by
  upstream modules are caught too. On every NixOS host,
  `home-manager.backupFileExtension` and `home-manager.backupCommand` are
  null. In every home (each `home-manager.users.<user>` and each
  `homeConfigurations.<home>`), no `home.file.<n>.force` is set; `xdg.*File`
  feed into `home.file`, so that covers them. An upstream module that sets
  `force` is either worked around or listed in the check's allowlist, with
  the reason.

Checks fail at evaluation (`assert`/`throw`), so `--no-build` runs them all.
A rule worth writing down is worth a check.

## Modules

```nix
# Git, lazygit and gh. Args: none.
{
  flake.modules.homeManager.git = {pkgs, ...}: {
    programs.git.enable = true;
    programs.lazygit.enable = true;
    home.packages = [pkgs.gh];
  };
}
```

- One file, one `flake.modules.<class>.<name>`, set directly. No wrapping
  `let`, no `config.flake.*` plumbing, no lines adding it to a host or user.
- Independent of where it runs. A module never names a user, host, domain,
  address or secret, and never reads `infra`. A value that changes between
  consumers is a module arg with no default (`{tailnet, ...}:`), named on the
  first line. The consumer passes it via `specialArgs`/`extraSpecialArgs`.
- Prefer the upstream option to a new arg. If NixOS or Home Manager already
  has an option for the value, the host or user sets it directly.
- Importing enables it. No `enable` options, no `mkIf` gates. Use
  `mkDefault` only on values consumers tune (packages, images, models).
- Modules never import each other. They may import the upstream module they
  wrap (`inputs.sops-nix.nixosModules.sops`). An upstream module that several
  modules need (like Home Manager's NixOS module) gets its own module, which
  hosts import once.
- Packages that nixpkgs lacks go in `packages/<name>.nix` (or
  `<name>/package.nix`) as plain derivations. They're loaded by
  `lib.filesystem.packagesFromDirectoryRecursive`. Modules use them as
  `self.packages.${pkgs.stdenv.hostPlatform.system}.<name>`. Never call
  `callPackage` outside that one loader.
- Don't reinvent the wheel. Before writing anything, look for an upstream
  option, module, nixpkgs function or mature flake that already does it
  (`sops.templates` for env files, `pkgs.writers` for scripts).
- Code should read as config. Computation that would clutter a file becomes a
  helper in its own file (see below). Helpers are never defined inline.

## Helpers

```nix
# mergeJson: a JSON file's contents with an attrset merged over them.
#   file   path to a JSON file
#   attrs  values that win over the file's, merged recursively
# Returns the merged attrset.
#   mergeJson ./settings.json {theme = "dark";}
{lib, ...}: {
  _module.args.mergeJson = file: attrs: lib.recursiveUpdate (lib.importJSON file) attrs;
}
```

- Before writing a helper, check that upstream doesn't already do it.
- Make one only when it makes the calling code read as config, or removes
  real repetition. If the name needs "and", it is two helpers.
- Local by default: `_lib/<name>.nix` beside its one caller. It's a plain
  function that takes what it needs (`{lib}: ...`) and is imported by relative
  path.
- Shared when more than one file calls it, or it's general-purpose (like
  `mergeJson`): `lib/<name>.nix` sets `_module.args.<name>`, and flake-parts
  files take it as an arg. Move a local helper here when it gains a second
  caller.
- The contract on top is mandatory: every arg, the return value, and one
  example call. A reader should never need to open the body.
- Never `flake.lib`. The API is modules, plus the `packages` they use.
  Exporting a helper is a design decision for when it would help downstream
  use, and it starts in README.md.
- Helpers never name a host, user or domain; the example call in the
  contract may.

## Hosts, users and services

- Hosts, users and services are where everything specific lives: names,
  addresses, emails, hostIds, keys, `stateVersion`, `sops.secrets`, routes,
  and which modules they use. They read `infra` and build the args their
  modules take.
- `hosts/<host>/default.nix` sets `flake.nixosConfigurations.<host>` with
  `nixpkgs.lib.nixosSystem`. Its imports list every module, every
  `nixosModules.<user>` and every `nixosModules.<service>` on the machine.
- `services/<service>/default.nix` sets `flake.nixosModules.<service>`: the
  service's reusable module from `modules/nixos/services/`, plus this
  deployment's values, secrets and data files. The host running it imports
  it.
- The test for a service: copying `modules/nixos/services/<x>.nix` into
  another flake must work unchanged. Anything that would break that belongs
  in `services/<x>/`.
- `users/<user>/default.nix` sets `flake.homeModules.<user>` (and variants),
  the matching `flake.nixosModules.<user>` for NixOS hosts (Home Manager as a
  NixOS module, never standalone there), and
  `flake.homeConfigurations."<user>@<host>"` for non-NixOS machines.
- Configs that change often are files in `users/<user>/config/<app>/`, linked
  with `mkOutOfStoreSymlink`. Configs set once are Home Manager options.
- To adopt an Omarchy-managed config, move it (`mv`, never copy) into
  `users/<user>/config/<app>/`, then link it. Never manage an Omarchy config
  that hasn't been adopted this way.
- Home Manager must fail, not clobber, when a file is in its way. Never set
  `backupFileExtension`, pass `-b`, or use `force`. The checks catch the
  options; `-b` is a CLI flag, so it's on you.
- A new host, user or service is listed in `flake.nix` imports. A new host or
  user also gets a row in the README's Machines table.

## Secrets

Every secret lives in the root `secrets.yaml` (sops-nix) under
`<group>/<name>`. Hosts, users and services declare `sops.secrets` and pass
paths. Modules never do. Adding a host or standalone home adds its age key to `.sops.yaml`,
followed by `sops updatekeys secrets.yaml`. Never print or commit a decrypted
value.

## Style

- Fewest lines. Re-read every edit and collapse what fits on one line.
- Every file opens with a one-line description; module descriptions end with
  their args.
- Plain attrsets over functions. When a function earns its place, it is a
  `lib/` helper with a contract.
- A mature upstream module or flake beats hand-rolled code, even as a new
  input.
- Comments only where a reader would stumble. The why goes in the commit
  message.
- Alejandra formats. Blank lines between attrsets, never inside a list. Use
  dotted keys for one value, nesting for several.
- Packages come from Nix only. Never add mise, global npm/pip installs or AUR
  packages to solve a problem here.
