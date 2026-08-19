# gh-install

A [GitHub CLI](https://cli.github.com/) extension that downloads and installs
the [Multipass](https://github.com/canonical/multipass) package built by CI for
a given pull request.

Think of it as `gh co 5135`, but instead of checking out the code it installs
the package produced by that PR's CI run, for your current OS.

## Usage

```console
$ gh install 5135                # install the package built for PR 5135
$ gh install 5135 --download     # download the package only (no install)
$ gh install --run 123456789     # install from a specific workflow run
```

### Flags

| Flag               | Description                                          |
| ------------------ | ---------------------------------------------------- |
| `--run <run-id>`   | Use a specific workflow run instead of the PR head   |
| `--download`       | Download only; do not install                        |
| `--keep-cache`     | Do not prune old cached packages on this run         |
| `-h`, `--help`     | Show help                                            |

### Caching

Downloaded packages are cached in `~/.cache/gh-install/packages` (or
`$XDG_CACHE_HOME/gh-install/packages`). Repeating a command for the same
artifact reuses the cached file — artifacts are immutable after upload, so
the cache entry is validated against the artifact's `updated_at` timestamp.

The cache prunes itself: packages older than 7 days are deleted on each run
(CI artifacts expire server-side anyway, so old packages are dead weight).
Tune the retention with the `GH_INSTALL_CACHE_DAYS` environment variable, or
skip pruning for a run with `--keep-cache`.

### What it does

1. Resolves the head commit of the PR (`gh pr view`).
2. Finds the most recent successful `Dynamic CI` run for that commit (the
   `macos.yml` / `windows.yml` / `linux.yml` workflows are called from it via
   `workflow_call`, so the packages are artifacts of that run).
3. Picks the matching, non-expired package artifact (`.pkg`, `.msi`/`.exe`, or
   `.snap`).
4. Downloads it (`gh run download`) and installs it:
   - macOS: `sudo installer -pkg … -target /`
   - Windows: `msiexec /i …` (or runs the `.exe` installer)
   - Linux: `sudo snap install --dangerous …`

> [!NOTE]
> GitHub Actions artifacts expire after a retention period. If the package is
> gone, re-run CI on the PR (e.g. `gh run rerun`) to produce a fresh one.

## Installing the extension

```console
$ gh extension install <owner>/gh-install
```

Upgrade later with:

```console
$ gh extension upgrade --all
```

## Platform support

- **macOS** and **Linux**: works in any POSIX shell with `gh` authenticated.
- **Windows**: `gh` executes script extensions with the `sh.exe` bundled with
  Git for Windows, so this single bash script works there too. Run it from a
  terminal where UAC elevation is possible for `msiexec`.

## Requirements

- [GitHub CLI](https://cli.github.com/) (`gh`), authenticated
  (`gh auth login`) with access to the Multipass repository.
- macOS/Linux: `sudo` rights to install the package.
- Windows: Git for Windows (installed automatically with `gh`), and permission
  to install software.

## License

[MIT](LICENSE)
