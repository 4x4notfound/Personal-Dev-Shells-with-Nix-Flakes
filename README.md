# Personal Dev Shells with Nix Flakes

A single flake at `~/dev/shell` defines named, composable development environments. A `dev` command enters any of them from anywhere.

<img width="852" height="480" alt="Terminal" src="https://github.com/user-attachments/assets/7e37fe72-b2e9-4081-8a6c-d2c68003b986" />


## Prerequisites

- macOS with [Determinate Nix](https://determinate.systems/nix) installed. Flakes are enabled out of the box; `nix-channel` is deprecated and unused here.
- zsh as your login shell (the default on macOS).

## Directory layout

```
~/dev/shell/
├── flake.nix          # all shell definitions
├── flake.lock         # pinned nixpkgs revision (commit this)
├── requirements.txt   # Python tools for the ai shell
├── .venv/             # created on first entry (gitignore)
└── litellm.yaml        # LiteLLM gateway config (when you get there)

~/.local/bin/
└── dev                # the entry command
```

Important: the flake lives in a dedicated directory, never in `~`. A flake's source is its entire containing directory, hashed and copied into the Nix store on evaluation. Pointing a flake at your home directory means hashing everything you own, and on macOS it fails outright on `~/.Trash` with `Operation not permitted`.

## Step 1: Create the flake

`~/dev/shell/flake.nix`:

```nix
{
  description = "Personal dev shells, selectable via `dev <name>`";

  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixpkgs-unstable";

  outputs = { self, nixpkgs }:
    let
      systems = [ "aarch64-darwin" "x86_64-darwin" "x86_64-linux" "aarch64-linux" ];
      forAll = f: nixpkgs.lib.genAttrs systems (system: f nixpkgs.legacyPackages.${system});
    in
    {
      devShells = forAll (pkgs:
        rec {
          # default: common CLI tools, the base everything builds on
          default = pkgs.mkShell {
            buildInputs = with pkgs; [
              jq curl git ripgrep fd fzf tree watch
            ];
            shellHook = ''
              echo "[base] common CLI tools loaded"
            '';
          };

          # ai: default + Python/uv, litellm via requirements.txt
          ai = pkgs.mkShell {
            inputsFrom = [ default ];
            buildInputs = with pkgs; [ python312 uv ];
            shellHook = ''
              export SHELL_ROOT="$HOME/dev/shell"
              export VENV_DIR="$SHELL_ROOT/.venv"
              export REQS="$SHELL_ROOT/requirements.txt"
              export REQS_HASH_FILE="$VENV_DIR/.reqs-hash"

              if [ ! -d "$VENV_DIR" ]; then
                uv venv "$VENV_DIR" --python ${pkgs.python312}/bin/python
              fi
              source "$VENV_DIR/bin/activate"

              if [ -f "$REQS" ]; then
                current_hash=$(shasum -a 256 "$REQS" | cut -d' ' -f1)
                cached_hash=$(cat "$REQS_HASH_FILE" 2>/dev/null || true)
                if [ "$current_hash" != "$cached_hash" ]; then
                  echo "requirements.txt changed, syncing venv..."
                  uv pip install -r "$REQS"
                  echo "$current_hash" > "$REQS_HASH_FILE"
                fi
              fi

              export LITELLM_MASTER_KEY="''${LITELLM_MASTER_KEY:-test-1234}"
              echo "[ai] python + uv + venv active"
            '';
          };

          # jvm: default + Java 21 toolchain
          jvm = pkgs.mkShell {
            inputsFrom = [ default ];
            buildInputs = with pkgs; [ temurin-bin-21 maven gradle ];
            shellHook = ''
              export JAVA_HOME=${pkgs.temurin-bin-21}
              echo "[jvm] $(java -version 2>&1 | head -1)"
            '';
          };

          # node: default + Node/TypeScript
          node = pkgs.mkShell {
            inputsFrom = [ default ];
            buildInputs = with pkgs; [ nodejs_22 pnpm typescript ];
            shellHook = ''
              echo "[node] node $(node --version), pnpm $(pnpm --version)"
            '';
          };

          # docs: default + document and diagram tooling
          docs = pkgs.mkShell {
            inputsFrom = [ default ];
            buildInputs = with pkgs; [ pandoc graphviz d2 ];
            shellHook = ''
              echo "[docs] pandoc $(pandoc --version | head -1 | cut -d' ' -f2)"
            '';
          };
        });
    };
}
```

## Step 2: Python tools

The `ai` shell will install any dependencies listed in `~/dev/shell/requirements.txt`:

## Step 3: The `dev` command

`~/.local/bin/dev`:

```bash
#!/usr/bin/env bash
flake="$HOME/dev/shell"

if [ $# -le 1 ]; then
  exec nix develop "$flake#${1:-default}" --command zsh
else
  first="$1"; shift
  exec nix develop "$flake#$first" --command dev "$@"
fi
```

```bash
chmod +x ~/.local/bin/dev
```

In `~/.zshrc`, ensure the script is on PATH:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Two deliberate choices:

- It is a script, not a shell function. `nix develop` spawns bash by default, so a zsh function defined in `.zshrc` vanishes inside the shell. A script on PATH works at any nesting depth in any shell. If you previously defined a `dev()` function, delete it; zsh resolves functions before PATH.
- `--command zsh` runs the shellHook (capturing the environment), then hands you your normal zsh with your prompt, aliases, and completions. Without it you live in bare bash inside every shell.

## Usage

```bash
dev            # base CLI tools
dev ai         # base + python/uv + litellm
dev jvm        # base + Java 21, Maven, Gradle
dev node       # base + Node 22, pnpm, tsc
dev docs       # base + pandoc, graphviz, d2

dev ai jvm     # compound: ai with jvm nested inside
exit           # pops one layer; echo $SHLVL shows depth

nix flake show ~/dev/shell    # list all shells
nix flake update ~/dev/shell  # bump pinned nixpkgs
```

Compounding works three ways:

1. Ad hoc nesting: run `dev jvm` from inside `dev ai`. PATH entries prepend, the inner shell wins on conflicting variables, each `exit` pops one layer.
2. One command: `dev ai jvm` chains the nesting for you (last name wins on conflicts).
3. Declared combos: for recurring pairings, add a real shell to the flake. One eval, one cache entry, no PATH duplication:

```nix
aijvm = pkgs.mkShell { inputsFrom = [ ai jvm ]; };
```

## Extending

| Want                             | Do                                                                            |
| -------------------------------- | ----------------------------------------------------------------------------- |
| Python tool everywhere `ai` runs | Add a line to `requirements.txt`; next entry syncs                            |
| Binary/system tool in one shell  | Add to that shell's `buildInputs`                                             |
| Tool in every shell              | Add to `default`'s `buildInputs` (invalidates eval cache for all shells once) |
| New environment                  | Add a `mkShell` with `inputsFrom = [ default ]`                               |
| Per-machine secrets              | Export the env var before running `dev`; the `:-` defaults yield              |

## Optional: direnv auto-activation

If you want the `ai` environment to activate automatically when you `cd` into a specific project directory (including the VS Code integrated terminal), use direnv instead of or alongside `dev`:

```bash
nix profile install nixpkgs#direnv nixpkgs#nix-direnv
```

In `~/.zshrc`:

```bash
eval "$(direnv hook zsh)"
```

In `~/.config/direnv/direnvrc`:

```bash
source ~/.nix-profile/share/nix-direnv/direnvrc
```

In the project directory:

```bash
echo "use flake ~/dev/shell#ai" > .envrc
direnv allow
```

To silence direnv's environment diff (the wall of `+VAR` output), create `~/.config/direnv/direnv.toml`:

```toml
[global]
hide_env_diff = true
```

Keep the one-line status messages; the difference between `using cached dev shell` and `cache invalidated` is useful signal when an entry suddenly takes 30 seconds.

## Troubleshooting

**`error: cannot read directory "~/.Trash": Operation not permitted`**
The flake (or an `.envrc`) is in your home directory. Move `flake.nix`, `flake.lock`, `requirements.txt`, and `.envrc` to `~/dev/shell`, then `rm -rf ~/.direnv ~/.venv`.

**`dev: command not found` inside a shell**
You are in bash inside `nix develop` and `dev` was a zsh function. Use the script form above and remove the function.

**`attribute 'nixpkgs' not found` with `nix-env -iA`**
You are on a flakes-first install (Determinate Nix). Use `nix profile install nixpkgs#<pkg>` instead of channel-based commands.

**First entry into a shell is slow**
Normal: one-time flake evaluation and downloads. Subsequent entries are cached. If every entry is slow, check what your `.zshrc` runs on startup; it executes once per nesting layer.

**Stale environment after editing the flake**
`nix develop` always re-evaluates on change. With direnv, watch for `Evaluating current devShell failed. Falling back to previous environment!` in the output; that means you are on the old environment and the new one has an error.
