# environments

Shared pixi environments for Juno (NIC). Every directory here is baked into the
Juno JupyterLab image at build time and appears as a kernel for **all users** on
first login — no copying of `pixi.toml` files around.

| Environment | Kernel name | Purpose |
|---|---|---|
| [Renderlocal](Renderlocal/pixi.toml) | `nebi-Renderlocal-default` | Panel / Bokeh / HoloViz rendering stack |

## How it works

Nebi cannot yet seed workspaces for all users (nebari-dev/nebi#529). As a bridge,
the image build in [equinor/juno-images](https://github.com/equinor/juno-images)
downloads this repo as a tarball (`JUNO_ENVS_REPO` / `JUNO_ENVS_REF` in its
Dockerfile), runs `pixi install --locked --all` in every `<name>/` directory and
installs the result under `/opt/juno-envs/<name>`. Nebi's kernel spec manager
(`nb_nebi_kernels.NebiKernelSpecManager`) lists them through
`NEBI_WORKSPACE_DISCOVERY_PATHS=/opt/juno-envs`:

- kernel name `nebi-<name>-<env>`, display name `<name>` (`<env>` is `default`
  unless the manifest defines `[environments]`);
- launched with `pixi run --frozen` inside `/opt/juno-envs/<name>`;
- the per-user nebi database is untouched, and a user's own nebi workspace with
  the same name shadows the baked one — so anyone can fork an environment by
  copying its `pixi.toml` into their own workspace and adding packages.

## Layout and rules

```
<name>/pixi.toml   # [workspace] name must equal the directory name
<name>/pixi.lock   # required — the build uses --locked and never re-solves
```

- `platforms` must include `linux-64` (add `osx-arm64`/`win-64` if people also
  want to run the same environment locally).
- `ipykernel` must be a dependency, otherwise there is no kernelspec and the
  build fails.
- Use `==1.2.3` for exact pins. Bare `1.2.3` is ambiguous and pixi rejects it.
- Lock files are format v7: pixi ≥ 0.68 is required locally; the image guards
  its own pixi version.
- Directories without a `pixi.toml` are ignored.
- Each environment adds roughly its installed size to the image (hundreds of MB
  to a few GB). Keep the set small and prefer broad, shared stacks over
  per-project variants.

## Adding or changing an environment

```sh
mkdir <name> && cd <name>
pixi init --platform linux-64 --platform osx-arm64   # creates pixi.toml
pixi add python ipykernel <packages...>                # updates pixi.toml + pixi.lock
```

Or edit `pixi.toml` by hand and run `pixi lock`. Commit **both** files and open a
PR; CI ([check.yml](.github/workflows/check.yml)) verifies that every lock file
is up to date, installs on linux-64 and contains a kernelspec — the same checks
the image build performs.

To ship it, bump `JUNO_ENVS_REF` in the juno-images Dockerfile to the new tag or
commit and rebuild the image. The image does not track `main` automatically.

## Using an environment outside Juno

The same files work on a laptop:

```sh
git clone https://github.com/equinor/environments
cd environments/Renderlocal
pixi install --locked
pixi run python -m ipykernel install --user --name renderlocal
```
